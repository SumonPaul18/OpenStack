# 📘 Kolla-Ansible Nova Compute Troubleshooting Guide
## Fixing "Nova Compute Failed to Register" - Libvirt SASL Authentication Error

> **Document Version:** 1.0  
> **Last Updated:** March 2026  
> **Author:** DevOps Infrastructure Team  
> **Target Audience:** System Administrators, Cloud Engineers, OpenStack Deployers  
> **Difficulty Level:** Intermediate to Advanced  
> **Estimated Reading Time:** 25-30 minutes  

---

## 📑 Table of Contents

```
1. 🎯 Executive Summary
2. 🖥️ Environment Overview
3. 🔍 Problem Statement
4. 📊 Error Analysis & Logs
5. 🧠 Root Cause Investigation
6. 🔧 Step-by-Step Troubleshooting Journey
7. ✅ Solution 1: Quick Fix (Lab/Testing)
8. ✅ Solution 2: Permanent Fix (Production)
9. 🔐 Understanding Libvirt SASL Authentication
10. 🧪 Verification & Testing Procedures
11. 🛡️ Prevention & Best Practices
12. 🔄 Related Topics & Advanced Scenarios
13. ❓ Frequently Asked Questions (FAQ)
14. 📚 Glossary of Terms
15. 🔗 References & Further Reading
16. 📝 Appendix: Command Reference Sheet
```

---

## 1. 🎯 Executive Summary

### What This Guide Covers

This comprehensive guide documents a real-world troubleshooting scenario encountered during an OpenStack deployment using **Kolla-Ansible**. The specific issue was:

> **"The Nova compute service failed to register itself on the following hosts: controller"**

This error occurred during the `kolla-ansible deploy` playbook execution, specifically in the `nova-cell` role. After systematic investigation, the root cause was identified as **Libvirt SASL Authentication Failure** - where the `nova-compute` container could not authenticate with the `nova-libvirt` container due to corrupted or mismatched credential files.

### Why This Matters

- Nova Compute is the core service that manages virtual machine lifecycle in OpenStack
- If nova-compute cannot register, you cannot launch, manage, or monitor virtual machines
- This is a common issue in Kolla-Ansible deployments, especially in all-in-one or multi-node setups
- Understanding this troubleshooting flow helps you resolve similar authentication issues in other OpenStack services

### Key Takeaways

1. **Always verify binary config files** like `passwd.db` using `strings`, not `cat`
2. **Container startup order matters** - libvirt must be ready before compute connects
3. **Password synchronization** between `passwords.yml`, `auth.conf`, and `passwd.db` is critical
4. **Lab vs Production settings** - SASL can be disabled for testing but must be enabled for production
5. **Systematic logging and verification** saves hours of guesswork

---

## 2. 🖥️ Environment Overview

### Deployment Architecture

```
┌─────────────────────────────────────────┐
│           Controller Node                │
│  (All-in-One: Control + Compute)        │
├─────────────────────────────────────────┤
│  • OS: Ubuntu 24.04 LTS (Noble)         │
│  • Kolla-Ansible Version: 2025.1        │
│  • OpenStack Release: 2025.1 (Caracal)  │
│  • Container Runtime: Docker            │
│  • Virtualization: KVM/QEMU via Libvirt │
│  • Network: Single NIC, Bridge Mode     │
│  • Storage: Local Disk, LVM Optional    │
└─────────────────────────────────────────┘
```

### Hardware Specifications

| Component | Specification | Purpose |
|-----------|--------------|---------|
| CPU | 8+ cores, VT-x/AMD-V enabled | VM execution, nested virtualization support |
| RAM | 16GB+ minimum | Container memory, VM allocation |
| Storage | 100GB+ SSD | OS, container images, VM disks |
| Network | 1Gbps Ethernet | OpenStack internal & external traffic |

### Software Stack

```yaml
# /etc/kolla/globals.yml (Relevant Sections)
kolla_base_distro: "ubuntu"
kolla_install_type: "source"  # or "binary"
openstack_release: "2025.1"

# Nova Configuration
nova_compute_virt_type: "kvm"  # or "qemu" for testing
libvirt_enable_sasl: true      # Production: true, Lab: can be false

# Network Configuration
network_interface: "eth0"
tunnel_interface: "eth0"
```

### Pre-Deployment Checklist ✅

Before running `kolla-ansible deploy`, ensure:

```bash
# 1. Verify virtualization support
egrep -c '(vmx|svm)' /proc/cpuinfo
# Expected: Number > 0 (e.g., 16)

# 2. Check Docker service
systemctl status docker
# Expected: active (running)

# 3. Verify Kolla-Ansible virtual environment
source /opt/venv-kolla/bin/activate
# Expected: (venv-kolla) prefix in prompt

# 4. Validate inventory file
cat /etc/kolla/multinode  # or your inventory file
# Expected: [controller] section with correct hostname/IP

# 5. Generate passwords if not exists
kolla-genpwd
# Expected: /etc/kolla/passwords.yml created/updated
```

---

## 3. 🔍 Problem Statement

### The Scenario

You are deploying OpenStack using Kolla-Ansible on a single controller node (all-in-one setup). The deployment progresses through most services successfully:

- ✅ Keystone (Identity)
- ✅ Glance (Image)
- ✅ Neutron (Networking)
- ✅ Placement (Resource tracking)
- ❌ Nova (Compute) - **FAILS HERE**

### The Error Message

During the `nova-cell` role execution, the playbook shows:

```
TASK [nova-cell : Waiting for nova-compute services to register themselves]
FAILED - RETRYING: [localhost]: Waiting for nova-compute services to register themselves (20 retries left).
[... 20 retry attempts ...]
FAILED - RETRYING: [localhost]: Waiting for nova-compute services to register themselves (1 retries left).
ok: [localhost]

TASK [nova-cell : Fail if nova-compute service failed to register]
fatal: [localhost]: FAILED! => {"changed": false, "msg": "The Nova compute service failed to register itself on the following hosts: controller"}
```

### What This Means in Simple Terms

- Nova Compute service started but could not "check in" with the central Nova API
- Think of it like an employee who arrived at the office but forgot to sign the attendance register
- The service is running, but OpenStack doesn't know it's available to launch VMs
- Result: `openstack server create` commands will fail with "No valid host was found"

### Business Impact

| Impact Area | Consequence |
|-------------|-------------|
| 🚫 VM Launch | Users cannot create new virtual machines |
| 🚫 VM Management | Existing VMs cannot be resized, migrated, or managed |
| 🚫 Monitoring | Nova metrics and health checks show "down" status |
| 🚫 Automation | CI/CD pipelines depending on OpenStack fail |
| 🚫 User Experience | End users see errors when trying to use cloud resources |

---

## 4. 📊 Error Analysis & Logs

### Step 1: Check Container Status

First, verify if the nova-compute container is actually running:

```bash
# Check if nova_compute container exists and its status
docker ps -a | grep nova_compute

# Example output when running:
# 86f415cd1148   quay.io/openstack.kolla/nova-compute:2025.1-ubuntu-noble   "dumb-init --single-…"   19 minutes ago   Up 6 seconds (healthy)   nova_compute

# Example output when crashed:
# 86f415cd1148   quay.io/openstack.kolla/nova-compute:2025.1-ubuntu-noble   "dumb-init --single-…"   3 hours ago   Exited (1) 2 minutes ago   nova_compute
```

**Interpretation:**
- `Up X seconds (healthy)` = Container is running and passing health checks ✅
- `Exited (1)` = Container crashed with error code 1 ❌
- `Restarting` = Container is in a crash loop 🔄

### Step 2: Examine Nova Compute Logs

The most important diagnostic information is in the nova-compute logs:

```bash
# View real-time logs with timestamps
docker logs -tf nova_compute

# Or view last 100 lines for recent errors
docker logs --tail 100 nova_compute

# Or filter for error patterns
docker logs nova_compute 2>&1 | grep -i "error\|fail\|authentication"
```

**Critical Error Pattern Found:**

```
2026-03-16 09:18:54.415 7 ERROR oslo_service.backend.eventlet.service
nova.exception.HypervisorUnavailable: Connection to the hypervisor is broken on host

2026-03-16 09:18:54.424 7 INFO nova.virt.libvirt.driver
Connection event '0' reason 'Failed to connect to libvirt: authentication failed: authentication failed'

2026-03-16 09:19:13.915 7 ERROR nova.virt.libvirt.host
Connection to libvirt failed: authentication failed: authentication failed: 
libvirt.libvirtError: authentication failed: authentication failed
```

**Decoding the Error:**

| Log Segment | Meaning |
|-------------|---------|
| `HypervisorUnavailable` | Nova cannot reach the virtualization layer (libvirt) |
| `authentication failed: authentication failed` | SASL password verification failed (note the double message) |
| `libvirt.libvirtError` | The error originates from the libvirt library, not Nova itself |

### Step 3: Check Libvirt Container Logs

Since the error mentions libvirt, also check the nova-libvirt container:

```bash
# Check libvirt container status
docker ps -a | grep nova_libvirt

# View libvirt logs
docker logs -tf nova_libvirt

# Filter for SASL-related messages
docker logs nova_libvirt 2>&1 | grep -i "sasl\|auth\|passwd"
```

### Step 4: Verify OpenStack Service Registration

Check if Nova services are visible to OpenStack:

```bash
# Load admin credentials
source /etc/kolla/admin-openrc.sh

# List all nova services
openstack compute service list

# Filter for nova-compute only
openstack compute service list --service nova-compute

# Example problematic output:
# +----+----------------+------------+----------+---------+-------+----------------------------+
# | ID | Binary         | Host       | Zone     | Status  | State | Updated At                 |
# +----+----------------+------------+----------+---------+-------+----------------------------+
# | 1  | nova-compute   | controller | nova     | disabled| down  | 2026-03-16T08:00:00.000000 |
# +----+----------------+------------+----------+---------+-------+----------------------------+
```

**Key Indicators:**
- `Status: disabled` = Service manually disabled (not our issue)
- `Status: enabled` + `State: down` = Service registered but not responding (our issue)
- `State: up` + `:-)` emoji = Service healthy ✅

---

## 5. 🧠 Root Cause Investigation

### Understanding Libvirt SASL Authentication

Before diving deeper, let's understand what SASL authentication is and why it matters:

```
┌─────────────────┐     SASL Auth      ┌─────────────────┐
│  nova-compute   │◄──────────────────►│  nova-libvirt   │
│   Container     │   TCP:16509        │   Container     │
│                 │   Username: nova   │                 │
│  auth.conf      │   Password: ****   │  passwd.db      │
│  (client creds) │◄──────────────────►│  (server creds) │
└─────────────────┘   Verify Match     └─────────────────┘
```

**How It Works:**

1. `nova-compute` wants to manage VMs → needs to talk to libvirt
2. Libvirt is configured to require SASL authentication over TCP (port 16509)
3. `nova-compute` sends credentials from `/var/lib/nova/.config/libvirt/auth.conf`
4. `nova-libvirt` checks these credentials against `/etc/libvirt/passwd.db`
5. If they match → connection allowed; If not → `authentication failed`

### Potential Root Causes (Ranked by Likelihood)

#### 🔴 Cause #1: Corrupted passwd.db File (Most Common)

**What Happened:**
The `/etc/libvirt/passwd.db` file is a **binary database file** (not plain text). During container startup or config copy, this file can get:
- Partially written (race condition)
- Encoding corruption (binary treated as text)
- Permission issues (wrong owner/group)

**Evidence from Our Case:**
```bash
# When we checked with cat (wrong way):
docker exec nova_libvirt cat /etc/libvirt/passwd.db

# Output showed garbage characters:
# a      _,▒▒9▒
# _,▒▒9▒TQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVsenovacontrolleruserPassword
#       ▒эh^
```

**The Correct Way to Check:**
```bash
# Use strings command for binary files:
docker exec nova_libvirt strings /etc/libvirt/passwd.db

# Expected clean output:
# nova@openstack:jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse
```

#### 🟡 Cause #2: Password Mismatch Between Files

**What Happened:**
Three places store the SASL password, and they must all match:

| File Location | Container | Purpose |
|--------------|-----------|---------|
| `/etc/kolla/passwords.yml` | Host | Source of truth, used by Kolla-Ansible |
| `/var/lib/nova/.config/libvirt/auth.conf` | nova-compute | Client credentials for connecting |
| `/etc/libvirt/passwd.db` | nova-libvirt | Server credentials for verification |

**How Mismatch Occurs:**
- `passwords.yml` updated but `kolla-ansible deploy` interrupted
- Manual edit to one file but not others
- Container restart without config re-sync

#### 🟡 Cause #3: Container Startup Race Condition

**What Happened:**
- `nova-compute` starts and immediately tries to connect to libvirt
- `nova-libvirt` is still initializing, SASL database not loaded yet
- Connection attempt fails → compute service cannot register

**Why It's Intermittent:**
- Depends on system load, disk I/O, network latency
- May work on retry, fail on first attempt
- Hard to reproduce consistently

#### 🟢 Cause #4: Network/Firewall Blocking Port 16509

**What Happened:**
- Libvirt SASL uses TCP port 16509 for authentication
- If firewall blocks this port, connection fails
- Less common in Docker/Kolla setups (containers share network namespace)

**Verification:**
```bash
# Check if libvirt is listening on port 16509
docker exec nova_libvirt netstat -tlnp | grep 16509

# Expected output:
# tcp   0   0.0.0.0:16509   0.0.0.0:*   LISTEN   1234/libvirtd

# Test connectivity from nova-compute container
docker exec nova_compute bash -c "echo 'QUIT' | nc -v controller 16509"
# Expected: Connection to controller 16509 port [tcp/*] succeeded!
```

### Diagnostic Decision Tree

```
Start: Nova compute failed to register
│
├─► Check container status: docker ps | grep nova_compute
│   │
│   ├─► Container Exited/Crashed ──► Check logs: docker logs nova_compute
│   │   │
│   │   ├─► "authentication failed" ──► Go to SASL troubleshooting (this guide)
│   │   ├─► "Connection refused" ──► Check libvirt container & port 16509
│   │   ├─► "No valid host" ──► Check virt_type (kvm vs qemu)
│   │   └─► Other error ──► Search logs for specific message
│   │
│   └─► Container Running (healthy) ──► Check service registration
│       │
│       ├─► openstack compute service list shows "down"
│       │   │
│       │   ├─► Check nova-compute logs for auth errors
│       │   └─► Check nova-libvirt logs for auth errors
│       │
│       └─► Service shows "up" ──► Issue may be resolved, retry deployment
│
└─► Proceed to solutions below
```

---

## 6. 🔧 Step-by-Step Troubleshooting Journey

### Phase 1: Initial Assessment (First 5 Minutes)

**Goal:** Confirm the error pattern and gather baseline information.

```bash
# 1. Activate Kolla-Ansible virtual environment
source /opt/venv-kolla/bin/activate
# Prompt should show: (venv-kolla) root@controller:~#

# 2. Check the failed playbook recap (if still in terminal)
# Look for: "failed=1" in PLAY RECAP section

# 3. Verify nova container status
docker ps -a | grep -E "nova_compute|nova_libvirt"

# 4. Quick log scan for authentication errors
docker logs --tail 50 nova_compute 2>&1 | grep -i "auth\|error"

# 5. Check OpenStack service status
source /etc/kolla/admin-openrc.sh
openstack compute service list --service nova-compute
```

**Decision Point:**
- If logs show `authentication failed` → Proceed to SASL fix (Phase 2)
- If logs show `Connection refused` → Check libvirt container & networking
- If logs show `HypervisorUnavailable` without auth error → Check virt_type & KVM support

### Phase 2: SASL Authentication Deep Dive (Next 10 Minutes)

**Goal:** Identify which SASL component is misconfigured.

#### Step 2.1: Verify passwords.yml (Source of Truth)

```bash
# Check if SASL password exists in Kolla's password file
grep -A1 "libvirt_sasl" /etc/kolla/passwords.yml

# Expected output:
# libvirt_sasl_username: nova
# libvirt_sasl_password: jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse

# If missing, generate it:
kolla-genpwd
# This regenerates ALL passwords - use with caution in production
# For single password, manually add to passwords.yml with a strong value
```

#### Step 2.2: Inspect passwd.db in nova-libvirt Container

```bash
# Enter the nova-libvirt container
docker exec -it nova_libvirt bash

# Inside container: Check the SASL password database
# WRONG WAY (shows garbage for binary file):
cat /etc/libvirt/passwd.db

# RIGHT WAY (extracts readable strings):
strings /etc/libvirt/passwd.db

# Expected clean output:
# nova@openstack:jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse

# Verify file permissions (should be restricted):
ls -l /etc/libvirt/passwd.db
# Expected: -rw-r----- 1 root nova ...

# Exit container
exit
```

**Red Flags:**
- ❌ `strings` output is empty → File not created or completely corrupted
- ❌ `strings` shows multiple entries or garbage → File corrupted
- ❌ Password doesn't match `passwords.yml` → Sync issue
- ❌ Permissions are `644` or `777` → Security risk, may cause issues

#### Step 2.3: Verify auth.conf in nova-compute Container

```bash
# Enter the nova-compute container
docker exec -it nova_compute bash

# Inside container: Check the client authentication config
cat /var/lib/nova/.config/libvirt/auth.conf

# Expected content:
# [auth-libvirt-0]
# username=nova
# password=jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse

# Verify the password matches passwd.db from Step 2.2
# Exit container
exit
```

**Red Flags:**
- ❌ File doesn't exist → Config not copied during deployment
- ❌ Username is not `nova` → Wrong configuration
- ❌ Password doesn't match `passwd.db` → Authentication will fail
- ❌ File has extra whitespace or formatting issues → Parser may fail

#### Step 2.4: Test Libvirt Connectivity Manually

```bash
# From host, test if libvirt port is listening
docker exec nova_libvirt netstat -tlnp | grep 16509
# Expected: tcp  0.0.0.0:16509  LISTEN  [libvirtd]

# From nova-compute container, test TCP connection to libvirt
docker exec nova_compute bash -c "timeout 5 bash -c '</dev/tcp/controller/16509' && echo 'Port open' || echo 'Port closed'"

# Alternative using nc (netcat):
docker exec nova_compute bash -c "echo 'QUIT' | nc -v -w 3 controller 16509"
# Expected: Connection to controller 16509 port [tcp/*] succeeded!

# Advanced: Try virsh command from nova-compute (will fail auth if SASL wrong)
docker exec nova_compute virsh -c qemu+tcp://controller/system list
# If SASL works: Lists VMs (or empty list)
# If SASL fails: error: authentication failed
```

### Phase 3: Quick Verification Before Fix (2 Minutes)

```bash
# Create a simple diagnostic summary
echo "=== Nova SASL Diagnostic Summary ==="
echo ""
echo "[1] passwords.yml SASL entry:"
grep "libvirt_sasl_password" /etc/kolla/passwords.yml | cut -d: -f2 | xargs

echo ""
echo "[2] passwd.db content (via strings):"
docker exec nova_libvirt strings /etc/libvirt/passwd.db 2>/dev/null | grep nova@openstack || echo "NOT FOUND or CORRUPTED"

echo ""
echo "[3] auth.conf password (last 8 chars for comparison):"
docker exec nova_compute bash -c "grep password /var/lib/nova/.config/libvirt/auth.conf | cut -d= -f2 | rev | cut -c1-8 | rev" 2>/dev/null || echo "NOT FOUND"

echo ""
echo "[4] Libvirt port 16509 status:"
docker exec nova_libvirt netstat -tlnp 2>/dev/null | grep 16509 | awk '{print $4, $7}' || echo "NOT LISTENING"

echo ""
echo "[5] Nova compute service status:"
source /etc/kolla/admin-openrc.sh 2>/dev/null
openstack compute service list --service nova-compute -c Host -c State -f value 2>/dev/null || echo "OpenStack CLI not available"
```

**Interpret the Summary:**
- If [2] or [3] shows "NOT FOUND" → Config files missing/corrupted
- If passwords in [1], [2], [3] don't match → Sync issue
- If [4] shows "NOT LISTENING" → Libvirt not configured for TCP/SASL
- If [5] shows "down" → Service not registering (our original problem)

---

## 7. ✅ Solution 1: Quick Fix (Lab/Testing Environment)

### When to Use This Solution

✅ **Use this if:**
- You are in a learning, development, or testing environment
- You need to get OpenStack working quickly for demos or experiments
- Security is not a primary concern (isolated lab network)
- You plan to re-enable SASL later for production testing

❌ **Do NOT use this if:**
- This is a production or customer-facing environment
- Your OpenStack cloud is accessible from untrusted networks
- You are preparing for security compliance audits
- You want to learn proper SASL troubleshooting

### Step-by-Step: Disable SASL for Testing

#### Step 1: Update globals.yml

```bash
# Edit the Kolla-Ansible global configuration file
nano /etc/kolla/globals.yml
# Or use vi/vim: vi /etc/kolla/globals.yml

# Add or modify these lines (anywhere in the file):
libvirt_enable_sasl: false
nova_libvirt_enable_sasl: false

# Optional: Also disable for nova-compute explicitly (redundant but clear)
nova_compute_libvirt_sasl: false

# Save and exit:
# nano: Ctrl+X, then Y, then Enter
# vim: :wq then Enter
```

**What This Does:**
- Tells Kolla-Ansible to configure libvirt WITHOUT SASL authentication
- Libvirt will accept connections without password verification
- Simplifies troubleshooting by removing one variable

#### Step 2: Re-deploy Only Nova Services

```bash
# Ensure you're in the Kolla-Ansible virtual environment
source /opt/venv-kolla/bin/activate

# Re-deploy ONLY nova-related services (faster than full deploy)
# Replace <inventory> with your actual inventory file path
# Common values: multinode, allinone, or a custom file
kolla-ansible deploy -i /etc/kolla/multinode --limit controller --tags nova

# What this command does:
# -i /etc/kolla/multinode  → Use this inventory file
# --limit controller       → Only run on controller group hosts
# --tags nova             → Only execute nova-related tasks
```

**Expected Output:**
```
PLAY [Apply role nova] **********************************************************************

TASK [nova-cell : include_tasks] ************************************************************
skipping: [controller]

TASK [nova-cell : Flush handlers] ***********************************************************

RUNNING HANDLER [nova-cell : Restart nova-libvirt container] ********************************
changed: [controller]

RUNNING HANDLER [nova-cell : Restart nova-compute container] ********************************
changed: [controller]

# ... more tasks ...

TASK [nova-cell : Waiting for nova-compute services to register themselves] *****************
ok: [controller]

TASK [nova-cell : Fail if nova-compute service failed to register] **************************
skipping: [controller]  ← This is GOOD! Task skipped means no failure

PLAY RECAP **********************************************************************************
controller  : ok=XX  changed=XX  unreachable=0  failed=0  skipped=XX  rescued=0  ignored=0
```

**Key Success Indicator:** `failed=0` in the PLAY RECAP

#### Step 3: Verify the Fix

```bash
# Wait 30-60 seconds for services to stabilize

# Check container health
docker ps | grep -E "nova_compute|nova_libvirt"
# Both should show: Up X minutes (healthy)

# Check OpenStack service registration
source /etc/kolla/admin-openrc.sh
openstack compute service list --service nova-compute

# Expected successful output:
# +----+----------------+------------+----------+--------+-------+-------------------+
# | ID | Binary         | Host       | Zone     | Status | State | Updated At        |
# +----+----------------+------------+----------+--------+-------+-------------------+
# | 1  | nova-compute   | controller | nova     | enabled| up    | 2026-03-16T12:30:00 |
# +----+----------------+------------+----------+--------+-------+-------------------+
```

#### Step 4: Test Basic Nova Functionality

```bash
# Create a test flavor (if none exists)
openstack flavor create --id 1 --vcpus 1 --ram 512 --disk 1 m1.tiny

# List available images
openstack image list

# Try to create a minimal server (will fail at network if Neutron not ready, 
# but should pass the "no valid host" error if nova-compute is working)
openstack server create --flavor m1.tiny --image <image-id> --wait test-vm 2>&1 | head -20

# If you get "No valid host was found" → Nova still not working
# If you get network-related errors → Nova IS working, issue is elsewhere ✅
```

### Post-Fix: Document Your Change

```bash
# Add a note to your deployment log or README
echo "# Lab Deployment Note" >> /etc/kolla/DEPLOYMENT_NOTES.md
echo "- Date: $(date)" >> /etc/kolla/DEPLOYMENT_NOTES.md
echo "- SASL disabled for testing: libvirt_enable_sasl: false" >> /etc/kolla/DEPLOYMENT_NOTES.md
echo "- Plan: Re-enable SASL before production deployment" >> /etc/kolla/DEPLOYMENT_NOTES.md
```

---

## 8. ✅ Solution 2: Permanent Fix (Production Environment)

### When to Use This Solution

✅ **Use this if:**
- This is a production, staging, or customer environment
- Security compliance requires authentication between services
- You want to learn proper SASL configuration for future troubleshooting
- You need a robust, maintainable OpenStack deployment

### Step-by-Step: Fix SASL Authentication Properly

#### Step 1: Backup Current State (Safety First)

```bash
# Create a backup directory with timestamp
BACKUP_DIR="/root/kolla-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

# Backup critical config files
cp /etc/kolla/passwords.yml "$BACKUP_DIR/"
cp /etc/kolla/globals.yml "$BACKUP_DIR/"

# Backup container configs (optional but recommended)
docker exec nova_libvirt cat /etc/libvirt/passwd.db > "$BACKUP_DIR/passwd.db.backup" 2>/dev/null
docker exec nova_compute cat /var/lib/nova/.config/libvirt/auth.conf > "$BACKUP_DIR/auth.conf.backup" 2>/dev/null

echo "Backup created: $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

#### Step 2: Reset passwd.db in nova-libvirt Container

```bash
# Enter the nova-libvirt container
docker exec -it nova_libvirt bash

# Inside container: Remove the corrupted/incorrect passwd.db
rm -f /etc/libvirt/passwd.db

# Create a new, clean passwd.db file with correct permissions
touch /etc/libvirt/passwd.db
chmod 640 /etc/libvirt/passwd.db
chown root:nova /etc/libvirt/passwd.db

# Verify permissions
ls -l /etc/libvirt/passwd.db
# Expected: -rw-r----- 1 root nova ...

# Exit container temporarily to get the password from host
exit
```

#### Step 3: Add Correct Password Entry

```bash
# Get the SASL password from passwords.yml
SASL_PASS=$(grep "libvirt_sasl_password:" /etc/kolla/passwords.yml | awk '{print $2}')
echo "Using password: ${SASL_PASS:0:8}... (truncated for security)"

# Re-enter the nova-libvirt container
docker exec -it nova_libvirt bash

# Inside container: Add the password entry using saslpasswd2
# Format: saslpasswd2 -p -c -f <file> -u <realm> <username>
saslpasswd2 -p -c -f /etc/libvirt/passwd.db -u openstack nova <<< "$SASL_PASS"

# Verify the entry was created correctly
# Use strings to read the binary file
strings /etc/libvirt/passwd.db

# Expected output format:
# nova@openstack:jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse

# Double-check: Password after colon should match passwords.yml
# Exit container
exit
```

**Troubleshooting saslpasswd2:**
- If command not found: `apt-get update && apt-get install -y libsasl2-modules` (inside container)
- If permission denied: Ensure you're root in container (docker exec runs as root by default)
- If no output from strings: File may still be empty → repeat the saslpasswd2 command

#### Step 4: Verify auth.conf in nova-compute Container

```bash
# Check what password nova-compute is using
docker exec nova_compute cat /var/lib/nova/.config/libvirt/auth.conf

# Expected content:
# [auth-libvirt-0]
# username=nova
# password=jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse

# Compare the password value with:
# 1. /etc/kolla/passwords.yml on host
# 2. strings output from passwd.db in Step 3

# If passwords don't match, the issue is in config propagation:
# Option A: Re-run kolla-ansible deploy to sync configs
# Option B: Manually update auth.conf (not recommended, may be overwritten)
```

#### Step 5: Restart Containers in Correct Order

```bash
# Restart nova-libvirt FIRST (it provides the authentication service)
docker restart nova_libvirt

# Wait for libvirt to fully initialize (10-15 seconds)
sleep 12

# Then restart nova-compute (it will now authenticate successfully)
docker restart nova_compute

# Monitor the startup logs
docker logs -tf nova_compute | grep -E "Starting|Connected|ERROR|authentication"
```

**Expected Log Sequence:**
```
[timestamp] INFO nova.service [-] Starting compute node (version 31.2.1)
[timestamp] INFO nova.virt.libvirt.driver [-] Successfully connected to libvirt
[timestamp] INFO nova.compute.service [-] Service nova-compute updated
[timestamp] INFO nova.compute.service [-] Starting nova-compute node
```

**Red Flag Logs:**
- `authentication failed` → Password still mismatched
- `Connection refused` → Libvirt not listening on port 16509
- `HypervisorUnavailable` → KVM/QEMU driver issue, not auth

#### Step 6: Verify Service Registration

```bash
# Wait 30 seconds for service to register with Nova API
sleep 30

# Check OpenStack service status
source /etc/kolla/admin-openrc.sh
openstack compute service list --service nova-compute

# Expected successful output:
# +----+----------------+------------+----------+--------+-------+-------------------+
# | ID | Binary         | Host       | Zone     | Status | State | Updated At        |
# +----+----------------+------------+----------+--------+-------+-------------------+
# | 1  | nova-compute   | controller | nova     | enabled| up    | 2026-03-16T12:35:00 |
# +----+----------------+------------+----------+--------+-------+-------------------+

# If still showing "down", check:
# 1. nova-compute logs for new errors
# 2. Message queue connectivity (RabbitMQ)
# 3. Database connectivity (MariaDB)
```

#### Step 7: Test Libvirt Connection Manually (Optional but Recommended)

```bash
# From nova-compute container, test authenticated libvirt connection
docker exec nova_compute virsh -c qemu+tcp://controller/system list

# If SASL is working correctly:
# - Command succeeds (may show empty list if no VMs)
# - No authentication error

# If still failing:
# error: authentication failed: authentication failed
# → Return to Step 3, verify passwd.db content again
```

#### Step 8: Complete the Kolla-Ansible Deployment

```bash
# If you interrupted the original deploy, resume it
# Or run a targeted deploy to ensure all nova configs are applied
kolla-ansible deploy -i /etc/kolla/multinode --limit controller --tags nova

# Verify final status
kolla-ansible post-deploy  # If this is your final deployment step
```

---

## 9. 🔐 Understanding Libvirt SASL Authentication

### What Is SASL?

**SASL** = Simple Authentication and Security Layer

- A framework for adding authentication support to connection-based protocols
- Not a specific authentication method, but a way to plug in methods (PLAIN, DIGEST-MD5, GSSAPI, etc.)
- Used by Libvirt to secure remote connections over TCP

### Why Does Libvirt Use SASL?

```
Without SASL:
[nova-compute] --(unauthenticated TCP)--> [libvirt] --(control VMs)
                          ↓
              Anyone on network could control your VMs! ❌

With SASL:
[nova-compute] --(username+password via SASL)--> [libvirt] --(control VMs)
                          ↓
              Only authorized services can control VMs ✅
```

### SASL Configuration Files Explained

#### File 1: `/etc/libvirt/libvirtd.conf` (Server Configuration)

```bash
# View the relevant SASL settings
docker exec nova_libvirt grep -E "^(auth_tcp|sasl)" /etc/libvirt/libvirtd.conf

# Typical production settings:
auth_tcp = "sasl"              # Require SASL auth for TCP connections
sasl_allowed_username_list = ["nova@openstack"]  # Only allow this user
```

| Setting | Purpose | Common Values |
|---------|---------|--------------|
| `auth_tcp` | Authentication method for TCP connections | `"sasl"`, `"none"` |
| `listen_tcp` | Enable TCP listener | `1` (yes), `0` (no) |
| `tcp_port` | Port for TCP connections | `"16509"` (default) |
| `listen_addr` | Network interface to bind | `"0.0.0.0"` (all), `"192.168.1.10"` (specific) |

#### File 2: `/etc/libvirt/passwd.db` (Server Credentials)

```bash
# This is a binary SASL password database
# Created/managed by: saslpasswd2 command
# Format when read with strings: username@realm:password

# Example entry:
nova@openstack:jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse

# NEVER edit this file with a text editor!
# ALWAYS use: saslpasswd2 -p -c -f /etc/libvirt/passwd.db -u openstack nova
```

**Why Binary?**
- Prevents accidental exposure via `cat` or log files
- Allows SASL library to store additional metadata (salt, hash method, etc.)
- Standard format for Cyrus SASL library

#### File 3: `/var/lib/nova/.config/libvirt/auth.conf` (Client Credentials)

```ini
# Plain text INI-style config file
# Used by nova-compute to authenticate to libvirt

[auth-libvirt-0]
username=nova
password=jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse
```

| Field | Required | Description |
|-------|----------|-------------|
| `username` | Yes | SASL username (must match passwd.db) |
| `password` | Yes | SASL password (must match passwd.db) |
| `realm` | No | SASL realm (defaults to hostname if not specified) |

### SASL Authentication Flow (Step-by-Step)

```
1. nova-compute starts
   ↓
2. Reads auth.conf → gets username=nova, password=****
   ↓
3. Opens TCP connection to controller:16509
   ↓
4. Libvirt (nova-libvirt) requests SASL authentication
   ↓
5. nova-compute sends credentials via SASL protocol
   ↓
6. Libvirt checks passwd.db for username@realm:password
   ↓
7. IF match → Connection established, VM operations allowed ✅
   IF no match → Connection rejected, "authentication failed" ❌
   ↓
8. If connected, nova-compute registers with Nova API
   ↓
9. OpenStack shows nova-compute service as "up" and ready
```

### Common SASL Misconceptions

| Misconception | Reality |
|--------------|---------|
| "SASL is optional for security" | SASL is the PRIMARY auth mechanism for remote libvirt; without it, any network user could control your hypervisor |
| "I can use cat to read passwd.db" | passwd.db is binary; use `strings` command to view readable content |
| "Changing passwords.yml auto-updates containers" | Kolla-Ansible must re-deploy to propagate config changes to running containers |
| "SASL errors always mean wrong password" | Could also be: file corruption, permissions, network issues, or libvirt not listening |

---

## 10. 🧪 Verification & Testing Procedures

### Post-Fix Verification Checklist

Run these commands after applying either solution to confirm everything works:

```bash
# 1. Container Health Check
echo "[1] Container Status:"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep nova

# Expected: nova_compute and nova_libvirt show "Up" and "(healthy)"

# 2. SASL Configuration Validation
echo -e "\n[2] SASL passwd.db content:"
docker exec nova_libvirt strings /etc/libvirt/passwd.db 2>/dev/null | grep nova@openstack || echo "NOT FOUND"

echo -e "\n[3] Client auth.conf content:"
docker exec nova_compute grep -E "username|password" /var/lib/nova/.config/libvirt/auth.conf 2>/dev/null || echo "NOT FOUND"

# 3. Network Connectivity Test
echo -e "\n[4] Libvirt TCP Port (16509):"
docker exec nova_libvirt netstat -tlnp 2>/dev/null | grep 16509 | awk '{print "Listening on:", $4}' || echo "NOT LISTENING"

echo -e "\n[5] Connection Test from nova-compute:"
docker exec nova_compute timeout 3 bash -c '</dev/tcp/controller/16509' 2>/dev/null && echo "✓ Port reachable" || echo "✗ Port unreachable"

# 4. OpenStack Service Registration
echo -e "\n[6] Nova Service Status:"
source /etc/kolla/admin-openrc.sh 2>/dev/null
openstack compute service list --service nova-compute -c Host -c Binary -c Status -c State -f table 2>/dev/null || echo "OpenStack CLI error"

# 5. Libvirt Connection Test (Advanced)
echo -e "\n[7] Manual Libvirt Connection Test:"
docker exec nova_compute virsh -c qemu+tcp://controller/system list 2>&1 | head -5

# 6. End-to-End VM Launch Test (If Environment Ready)
echo -e "\n[8] Basic VM Creation Test:"
# Only run if you have images and networks configured
# openstack server create --flavor m1.tiny --image <id> --network <id> --wait test-sasl-fix 2>&1 | tail -10
```

### Expected Results Table

| Test | Expected Result | Failure Indicates |
|------|----------------|-------------------|
| Container Status | `Up (healthy)` | Container crash, resource issue |
| passwd.db content | `nova@openstack:****` | File corruption, config sync issue |
| auth.conf content | `username=nova`, matching password | Config propagation failure |
| Port 16509 listening | `0.0.0.0:16509 LISTEN` | Libvirt TCP config issue |
| Connection test | `✓ Port reachable` | Firewall, network, or libvirt not running |
| Nova service list | `Status: enabled, State: up` | Service registration failure |
| virsh connection | Lists VMs or empty list (no auth error) | SASL auth still failing |

### Monitoring After Fix

```bash
# Watch nova-compute logs for 2 minutes to ensure stability
docker logs -tf nova_compute | grep -E "INFO|ERROR" | tail -20

# Monitor OpenStack service status over time
watch -n 10 'source /etc/kolla/admin-openrc.sh && openstack compute service list --service nova-compute -c Host -c State -f value'

# Check for recurring authentication errors
docker logs nova_compute 2>&1 | grep -c "authentication failed"
# Expected: 0 (or very low number from initial attempts)
```

### Creating a Simple Health Check Script (For Future Use)

```bash
# Save this as /usr/local/bin/check-nova-sasl.sh for future troubleshooting
cat > /usr/local/bin/check-nova-sasl.sh << 'EOF'
#!/bin/bash
# Nova SASL Health Check Script
# Usage: ./check-nova-sasl.sh

echo "=== Nova SASL Health Check ==="
echo "Timestamp: $(date)"
echo ""

# Check containers
echo "[1] Container Status:"
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "nova_compute|nova_libvirt"
echo ""

# Check SASL files
echo "[2] passwd.db (last entry):"
docker exec nova_libvirt strings /etc/libvirt/passwd.db 2>/dev/null | tail -1 || echo "ERROR: Cannot read passwd.db"
echo ""

echo "[3] auth.conf password (masked):"
docker exec nova_compute bash -c "grep password /var/lib/nova/.config/libvirt/auth.conf 2>/dev/null | cut -d= -f2 | sed 's/./•/g'" || echo "ERROR: Cannot read auth.conf"
echo ""

# Check service
echo "[4] OpenStack Nova Service:"
source /etc/kolla/admin-openrc.sh 2>/dev/null
openstack compute service list --service nova-compute -c Host -c State -f value 2>/dev/null || echo "ERROR: OpenStack CLI failed"
echo ""

echo "=== Check Complete ==="
EOF

# Make it executable
chmod +x /usr/local/bin/check-nova-sasl.sh

# Run it anytime to quickly assess SASL health:
/usr/local/bin/check-nova-sasl.sh
```

---

## 11. 🛡️ Prevention & Best Practices

### Proactive Configuration Management

#### Best Practice #1: Always Use kolla-genpwd for Passwords

```bash
# When setting up a new deployment, generate passwords properly:
kolla-genpwd

# This ensures:
# - All required passwords are created
# - Passwords are strong and random
# - passwords.yml is properly formatted
# - No manual typos in sensitive credentials

# If you must change a single password:
# 1. Generate a strong password: openssl rand -hex 32
# 2. Edit passwords.yml carefully
# 3. Re-deploy affected services: kolla-ansible deploy --tags <service>
```

#### Best Practice #2: Validate Configs Before Deploy

```bash
# Before running deploy, validate your configuration:
kolla-ansible pre-deploy-checks -i /etc/kolla/multinode

# Check for common issues:
# - SSH connectivity to nodes
# - Docker version compatibility
# - Required Python packages
# - Network configuration

# Also manually review critical files:
grep -E "enable_sasl|virt_type" /etc/kolla/globals.yml
grep "libvirt_sasl" /etc/kolla/passwords.yml
```

#### Best Practice #3: Document Your Environment

```bash
# Create a deployment record:
cat > /etc/kolla/DEPLOYMENT_RECORD.md << EOF
# OpenStack Deployment Record

## Environment
- Date: $(date)
- Kolla-Ansible Version: $(kolla-ansible --version 2>/dev/null | head -1)
- OpenStack Release: 2025.1
- Deployment Type: All-in-One / Multi-Node

## Critical Settings
- libvirt_enable_sasl: true/false
- nova_compute_virt_type: kvm/qemu
- Network interface: eth0

## Known Issues & Workarounds
- [Date] SASL auth issue: Fixed by resetting passwd.db (see troubleshooting log)

## Recovery Procedures
- To reset SASL: See /root/kolla-troubleshooting/sasl-reset-guide.md
- Backup location: /root/kolla-backups/
EOF
```

### Operational Best Practices

#### Practice #1: Monitor Container Health

```bash
# Add to your monitoring system (Prometheus, Zabbix, etc.):
# Metric: container status for nova_compute and nova_libvirt
# Alert if: Status != "healthy" for > 2 minutes

# Simple cron-based check (add to crontab):
*/5 * * * * /usr/local/bin/check-nova-sasl.sh >> /var/log/nova-sasl-check.log 2>&1
```

#### Practice #2: Log Rotation and Retention

```bash
# Ensure Docker logs don't fill your disk:
# Edit /etc/docker/daemon.json:
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# Restart Docker to apply:
systemctl restart docker

# For Kolla-specific logs, configure logrotate:
cat > /etc/logrotate.d/kolla-nova << 'EOF'
/var/log/kolla/nova/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 nova nova
}
EOF
```

#### Practice #3: Regular Configuration Audits

```bash
# Monthly: Verify critical config files haven't drifted
echo "=== Monthly Config Audit ===" > /var/log/kolla-config-audit.log
date >> /var/log/kolla-config-audit.log

# Check passwords.yml integrity
md5sum /etc/kolla/passwords.yml >> /var/log/kolla-config-audit.log

# Check globals.yml for unexpected changes
grep -E "enable_sasl|virt_type" /etc/kolla/globals.yml >> /var/log/kolla-config-audit.log

# Verify container configs match host configs
docker exec nova_libvirt strings /etc/libvirt/passwd.db | grep nova@openstack >> /var/log/kolla-config-audit.log

echo "Audit complete" >> /var/log/kolla-config-audit.log
```

### Troubleshooting Preparedness

#### Create a Troubleshooting Toolkit

```bash
# Directory for troubleshooting resources
mkdir -p /root/kolla-troubleshooting

# Save common diagnostic commands as scripts:
cat > /root/kolla-troubleshooting/nova-sasl-diag.sh << 'EOF'
#!/bin/bash
# Nova SASL Diagnostic Script
echo "Collecting Nova SASL diagnostics..."
echo "Timestamp: $(date)"
echo ""

echo "=== Container Status ==="
docker ps -a | grep -E "nova_compute|nova_libvirt"
echo ""

echo "=== passwd.db Content ==="
docker exec nova_libvirt strings /etc/libvirt/passwd.db 2>/dev/null
echo ""

echo "=== auth.conf Content ==="
docker exec nova_compute cat /var/lib/nova/.config/libvirt/auth.conf 2>/dev/null
echo ""

echo "=== Libvirt Port Status ==="
docker exec nova_libvirt netstat -tlnp | grep 16509
echo ""

echo "=== Recent nova-compute Logs (auth-related) ==="
docker logs --tail 100 nova_compute 2>&1 | grep -i "auth\|error" | tail -20
echo ""

echo "=== OpenStack Service Status ==="
source /etc/kolla/admin-openrc.sh 2>/dev/null
openstack compute service list --service nova-compute 2>/dev/null
echo ""

echo "Diagnostics complete. Save this output for support tickets."
EOF
chmod +x /root/kolla-troubleshooting/nova-sasl-diag.sh
```

#### Document Your Fix Process

```bash
# After resolving an issue, document it:
cat > /root/kolla-troubleshooting/ISSUE-2026-03-16-nova-sasl.md << 'EOF'
# Issue: Nova Compute Failed to Register - SASL Auth Error

## Date
2026-03-16

## Symptoms
- kolla-ansible deploy failed at nova-cell role
- Error: "The Nova compute service failed to register itself"
- Logs showed: "authentication failed: authentication failed"

## Root Cause
- /etc/libvirt/passwd.db file was corrupted (binary file with garbage characters)
- Password in passwd.db did not match passwords.yml

## Resolution Steps
1. Verified passwords.yml had valid libvirt_sasl_password
2. Removed corrupted passwd.db from nova_libvirt container
3. Recreated passwd.db using saslpasswd2 command
4. Verified auth.conf in nova_compute matched the password
5. Restarted containers in order: nova_libvirt → wait → nova_compute
6. Confirmed service registration with openstack compute service list

## Prevention
- Always use strings command to inspect passwd.db
- Ensure kolla-ansible deploy completes fully after password changes
- Consider disabling SASL for lab environments during initial testing

## References
- Kolla-Ansible Documentation: https://docs.openstack.org/kolla-ansible/latest/
- Libvirt SASL Guide: https://libvirt.org/auth.html
EOF
```

---

## 12. 🔄 Related Topics & Advanced Scenarios

### Topic 1: Nested Virtualization and virt_type

**Scenario:** You're running OpenStack inside a VM (e.g., Proxmox, VMware, VirtualBox)

**Issue:** KVM may not be available, causing `HypervisorUnavailable` errors

**Solution:**

```bash
# Check if KVM is available on your host
egrep -c '(vmx|svm)' /proc/cpuinfo
# If result is 0 → KVM not available

# In /etc/kolla/globals.yml, set:
nova_compute_virt_type: qemu

# Then re-deploy:
kolla-ansible deploy -i <inventory> --tags nova

# Note: qemu is software emulation → slower than kvm, but works without hardware virtualization
```

**When to Use Each:**

| virt_type | Use When | Performance |
|-----------|----------|-------------|
| `kvm` | Bare metal or nested virt enabled | ⚡ Fast (hardware acceleration) |
| `qemu` | VM without nested virt, testing | 🐌 Slower (software emulation) |
| `lxc` | Container-based workloads | ⚡ Fast, but limited OS support |

### Topic 2: Multi-Node Deployments

**Scenario:** You have separate controller and compute nodes

**Additional Considerations:**

```bash
# 1. Ensure network connectivity between nodes:
# From compute node, test connection to controller:16509
nc -zv controller 16509

# 2. SASL passwords must be synchronized across all nodes:
# passwords.yml on controller should be copied to compute nodes
# Or use a shared configuration management system

# 3. Firewall rules must allow port 16509:
# On controller (if using firewalld):
firewall-cmd --add-port=16509/tcp --permanent
firewall-cmd --reload

# 4. Hostname resolution must work:
# All nodes should resolve "controller" to the correct IP
# Add to /etc/hosts if DNS not configured:
# 192.168.1.10 controller
```

### Topic 3: Upgrading Kolla-Ansible Versions

**Scenario:** You're upgrading from 2024.1 to 2025.1

**SASL-Specific Considerations:**

```bash
# 1. Backup before upgrade:
kolla-ansible backup -i <inventory>

# 2. Review release notes for SASL-related changes:
# https://docs.openstack.org/releasenotes/kolla-ansible/

# 3. After upgrade, verify SASL config:
grep "libvirt_enable_sasl" /etc/kolla/globals.yml
# Default may have changed between versions

# 4. Re-generate passwords if schema changed:
kolla-genpwd --keep-existing-passwords

# 5. Test in staging before production upgrade
```

### Topic 4: Integrating with External Identity Providers

**Scenario:** You want to use LDAP/Active Directory for libvirt SASL auth

**Advanced Configuration:**

```bash
# 1. Install SASL LDAP module in nova-libvirt container:
docker exec nova_libvirt apt-get update
docker exec nova_libvirt apt-get install -y libsasl2-modules-ldap

# 2. Configure /etc/libvirt/sasl2/libvirt.conf:
docker exec -it nova_libvirt bash
cat > /etc/libvirt/sasl2/libvirt.conf << 'EOF'
pwcheck_method: auxprop
auxprop_plugin: ldap
EOF

# 3. Configure LDAP settings in /etc/libvirt/sasl2/libvirt.conf
# (Requires LDAP server details, bind DN, search base, etc.)

# 4. Update nova-compute auth.conf to use LDAP credentials

# ⚠️ This is complex and rarely needed for typical OpenStack deployments
# Most environments use the simple passwd.db method
```

### Topic 5: Debugging with Increased Verbosity

**When logs aren't detailed enough:**

```bash
# 1. Enable debug logging in nova.conf:
# Edit /etc/kolla/nova/nova.conf on host, add:
[DEFAULT]
debug = true
verbose = true

# 2. Re-deploy nova services:
kolla-ansible deploy -i <inventory> --tags nova

# 3. Check logs with debug output:
docker logs -tf nova_compute | grep -A5 -B5 "authentication"

# 4. For libvirt debugging:
docker exec nova_libvirt libvirtd --debug=3 --log-file=/var/log/libvirtd-debug.log

# ⚠️ Debug logging generates large logs → disable after troubleshooting
```

---

## 13. ❓ Frequently Asked Questions (FAQ)

### Q1: Can I completely disable SASL in production?

**A:** Technically yes, but **strongly discouraged**.

```yaml
# In globals.yml:
libvirt_enable_sasl: false
```

**Risks:**
- Any user with network access to port 16509 can control your hypervisor
- Violates security best practices and compliance requirements (PCI-DSS, HIPAA, etc.)
- Makes your deployment vulnerable to insider threats and lateral movement

**Better Alternative:** Fix SASL properly using the permanent solution in this guide.

### Q2: Why does `cat` show garbage for passwd.db but `strings` works?

**A:** Because `passwd.db` is a **binary database file**, not plain text.

```bash
# Binary files contain non-printable characters, encoding metadata, etc.
# cat tries to display all bytes as text → shows garbage like ▒▒9▒

# strings extracts only printable character sequences:
strings /etc/libvirt/passwd.db
# Output: nova@openstack:jiwQTQWYxCiG4MShxr02lkGKVeRCTgdmtC3FvVse

# Other tools for binary files:
hexdump -C /etc/libvirt/passwd.db | head   # View raw hex
file /etc/libvirt/passwd.db                 # Identify file type
```

### Q3: How do I know if my deployment is using SASL?

**A:** Check multiple indicators:

```bash
# 1. Check globals.yml setting
grep "libvirt_enable_sasl" /etc/kolla/globals.yml
# true = SASL enabled, false = disabled

# 2. Check libvirtd.conf in container
docker exec nova_libvirt grep "auth_tcp" /etc/libvirt/libvirtd.conf
# auth_tcp = "sasl" → SASL enabled
# auth_tcp = "none" → SASL disabled

# 3. Check if passwd.db exists and has content
docker exec nova_libvirt ls -l /etc/libvirt/passwd.db
docker exec nova_libvirt strings /etc/libvirt/passwd.db
# File exists + has nova@openstack entry → SASL configured
```

### Q4: What if I forgot the SASL password?

**A:** Reset it using the proper procedure:

```bash
# 1. Generate a new strong password
NEW_PASS=$(openssl rand -hex 32)
echo "New SASL password: $NEW_PASS"  # Save this securely!

# 2. Update passwords.yml
sed -i "s/^libvirt_sasl_password:.*/libvirt_sasl_password: $NEW_PASS/" /etc/kolla/passwords.yml

# 3. Reset passwd.db in container (as shown in Solution 2, Step 3)
docker exec -it nova_libvirt bash
rm -f /etc/libvirt/passwd.db
touch /etc/libvirt/passwd.db && chmod 640 /etc/libvirt/passwd.db && chown root:nova /etc/libvirt/passwd.db
saslpasswd2 -p -c -f /etc/libvirt/passwd.db -u openstack nova <<< "$NEW_PASS"
exit

# 4. Re-deploy nova services to sync auth.conf
kolla-ansible deploy -i <inventory> --tags nova

# 5. Verify the fix
openstack compute service list --service nova-compute
```

### Q5: Can I use this guide for other OpenStack services with SASL?

**A:** Yes! The concepts apply to other services that use libvirt or SASL:

| Service | Similar Auth Mechanism | Adaptation Needed |
|---------|----------------------|------------------|
| Nova (this guide) | libvirt SASL over TCP:16509 | None - this is the reference |
| Neutron (with OVS) | OVSDB authentication | Check OVS auth config, not libvirt |
| Cinder (with LVM) | Usually no SASL | Not applicable |
| Ironic (bare metal) | IPMI/BMC authentication | Different protocol, similar troubleshooting approach |

**General Troubleshooting Pattern:**
1. Identify the authentication protocol (SASL, TLS, token, etc.)
2. Locate client credentials config (auth.conf, secrets, etc.)
3. Locate server credentials config (passwd.db, LDAP, etc.)
4. Verify they match and are properly formatted
5. Test connectivity and authentication manually
6. Restart services in correct order

### Q6: How long should I wait for nova-compute to register?

**A:** Typical timing expectations:

```bash
# After container restart:
# - 0-10 seconds: Container starts, reads config
# - 10-30 seconds: Connects to libvirt, authenticates
# - 30-60 seconds: Registers with Nova API, updates service status

# If still "down" after 2 minutes:
# 1. Check logs for errors: docker logs nova_compute | tail -50
# 2. Verify RabbitMQ connectivity: docker exec nova_compute nc -zv controller 5672
# 3. Verify database connectivity: docker exec nova_compute mysql -h controller -u nova -p<pass> -e "SELECT 1"

# Kolla-Ansible default retry settings:
# - Waiting for registration: 20 retries × 30 seconds = 10 minutes total
# - You can adjust in globals.yml if needed:
nova_service_register_timeout: 300  # seconds
```

---

## 14. 📚 Glossary of Terms

| Term | Definition | Relevance to This Guide |
|------|-----------|------------------------|
| **Kolla-Ansible** | OpenStack deployment tool using Docker containers and Ansible | The framework where this error occurred |
| **Nova** | OpenStack compute service that manages virtual machines | The service that failed to register |
| **Libvirt** | API for managing virtualization platforms (KVM, QEMU, etc.) | Nova uses libvirt to control VMs |
| **SASL** | Simple Authentication and Security Layer framework | The authentication method that was failing |
| **passwd.db** | Binary SASL password database file | The corrupted file causing authentication failures |
| **auth.conf** | Plain text config file with client credentials | Nova-compute uses this to authenticate to libvirt |
| **Race Condition** | Timing-dependent bug where order of operations matters | Possible cause if containers start in wrong order |
| **All-in-One** | Single-node OpenStack deployment (control + compute) | The environment type in this troubleshooting scenario |
| **virt_type** | Nova setting: kvm (hardware) or qemu (software) virtualization | Related error if KVM not available |
| **Container Health** | Docker feature to monitor if a service is functioning | Used to verify nova-compute is truly running |

### Acronyms

- **API**: Application Programming Interface
- **Auth**: Authentication
- **DB**: Database
- **INI**: Configuration file format (like Windows .ini files)
- **INI**: Initialization (in startup context)
- **SASL**: Simple Authentication and Security Layer
- **TCP**: Transmission Control Protocol (network layer)
- **VM**: Virtual Machine

---

## 15. 🔗 References & Further Reading

### Official Documentation

- **Kolla-Ansible Guide**: https://docs.openstack.org/kolla-ansible/latest/
- **Nova Administrator Guide**: https://docs.openstack.org/nova/latest/admin/
- **Libvirt Authentication**: https://libvirt.org/auth.html
- **Cyrus SASL Documentation**: https://www.cyrusimap.org/sasl/

### Community Resources

- **OpenStack Ask Forum**: https://ask.openstack.org/
- **Kolla-Ansible GitHub**: https://github.com/openstack/kolla-ansible
- **OpenStack IRC Channels**: #openstack, #kolla on libera.chat

### Related Troubleshooting Guides

- "Fixing Neutron Agent Registration Failures in Kolla-Ansible"
- "Resolving RabbitMQ Connection Issues in OpenStack"
- "Debugging MariaDB Connectivity in Containerized OpenStack"

### Tools Mentioned

- **Docker CLI**: https://docs.docker.com/engine/reference/commandline/cli/
- **OpenStack Client**: https://docs.openstack.org/python-openstackclient/latest/
- **Ansible**: https://docs.ansible.com/
- **saslpasswd2**: Part of Cyrus SASL package (`libsasl2-modules`)

---

## 16. 📝 Appendix: Command Reference Sheet

### Quick Diagnostic Commands

```bash
# Check nova container status
docker ps -a | grep -E "nova_compute|nova_libvirt"

# View nova-compute logs (last 50 lines)
docker logs --tail 50 nova_compute

# Search logs for authentication errors
docker logs nova_compute 2>&1 | grep -i "auth\|error"

# Check OpenStack nova service status
source /etc/kolla/admin-openrc.sh
openstack compute service list --service nova-compute

# Verify SASL passwd.db content
docker exec nova_libvirt strings /etc/libvirt/passwd.db

# Verify client auth.conf content
docker exec nova_compute cat /var/lib/nova/.config/libvirt/auth.conf

# Test libvirt TCP port connectivity
docker exec nova_compute nc -zv controller 16509

# Manual libvirt connection test
docker exec nova_compute virsh -c qemu+tcp://controller/system list
```

### Fix Commands (SASL Reset)

```bash
# Remove corrupted passwd.db
docker exec nova_libvirt rm -f /etc/libvirt/passwd.db

# Create new passwd.db with correct permissions
docker exec nova_libvirt bash -c "touch /etc/libvirt/passwd.db && chmod 640 /etc/libvirt/passwd.db && chown root:nova /etc/libvirt/passwd.db"

# Add password entry (replace YOUR_PASSWORD with actual value)
SASL_PASS="YOUR_PASSWORD"
docker exec nova_libvirt bash -c "saslpasswd2 -p -c -f /etc/libvirt/passwd.db -u openstack nova <<< '$SASL_PASS'"

# Verify the entry
docker exec nova_libvirt strings /etc/libvirt/passwd.db

# Restart containers in order
docker restart nova_libvirt
sleep 10
docker restart nova_compute
```

### Configuration Commands

```bash
# Edit globals.yml to disable SASL (lab only)
echo "libvirt_enable_sasl: false" >> /etc/kolla/globals.yml
echo "nova_libvirt_enable_sasl: false" >> /etc/kolla/globals.yml

# Re-deploy nova services after config change
kolla-ansible deploy -i /etc/kolla/multinode --limit controller --tags nova

# Generate new passwords (use with caution)
kolla-genpwd

# Validate pre-deployment checks
kolla-ansible pre-deploy-checks -i /etc/kolla/multinode
```

### Verification Commands

```bash
# Full health check sequence
echo "=== Nova SASL Health Check ==="
docker ps | grep nova | awk '{print $1, $7}'
docker exec nova_libvirt strings /etc/libvirt/passwd.db | grep nova@openstack
docker exec nova_compute grep password /var/lib/nova/.config/libvirt/auth.conf
source /etc/kolla/admin-openrc.sh && openstack compute service list --service nova-compute -c State -f value
echo "=== Check Complete ==="

# Monitor service registration in real-time
watch -n 5 'source /etc/kolla/admin-openrc.sh && openstack compute service list --service nova-compute -c Host -c State'

# Check for recurring errors over time
docker logs -tf nova_compute | grep -c "authentication failed"
```

### Backup & Recovery Commands

```bash
# Create timestamped backup
BACKUP_DIR="/root/kolla-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp /etc/kolla/passwords.yml "$BACKUP_DIR/"
cp /etc/kolla/globals.yml "$BACKUP_DIR/"
docker exec nova_libvirt cat /etc/libvirt/passwd.db > "$BACKUP_DIR/passwd.db" 2>/dev/null

# Restore from backup (use with extreme caution)
# cp /root/kolla-backup-*/passwords.yml /etc/kolla/
# docker exec -i nova_libvirt bash -c "cat > /etc/libvirt/passwd.db" < /root/kolla-backup-*/passwd.db
# docker restart nova_libvirt nova_compute
```

---

## 🎯 Final Thoughts

### Key Lessons from This Troubleshooting Journey

1. **Binary Files Require Special Handling**: Always use `strings` for `passwd.db`, never `cat`
2. **Authentication is Chain of Trust**: passwords.yml → auth.conf → passwd.db must all match
3. **Timing Matters**: Restart libvirt before compute to avoid race conditions
4. **Lab vs Production Trade-offs**: Disabling SASL speeds up testing but compromises security
5. **Documentation Saves Time**: Record your fixes to accelerate future troubleshooting

### When to Ask for Help

If you've followed this guide and still face issues:

```bash
# Collect this information before reaching out:
1. Full output of: /usr/local/bin/check-nova-sasl.sh
2. Last 100 lines of: docker logs nova_compute
3. Your globals.yml SASL-related settings (mask passwords)
4. OpenStack version and Kolla-Ansible version
5. Deployment type (all-in-one vs multi-node)

# Share with:
- OpenStack IRC: #kolla on libera.chat
- Ask OpenStack: https://ask.openstack.org/
- GitHub Issues: https://github.com/openstack/kolla-ansible/issues
```

### Continuous Improvement

This guide is a living document. As you encounter new scenarios:

1. Update the FAQ with new questions
2. Add new verification commands to the reference sheet
3. Document environment-specific quirks in the Appendix
4. Share improvements with your team or the community

---

> **Remember**: Every error is a learning opportunity. The time you invest in understanding *why* something failed makes you a stronger engineer for the next challenge.

*Document End*  
*Total Word Count: ~4,500 words*  
*Estimated Troubleshooting Time Saved: 2-4 hours per incident*  
*Confidence Level After Reading: High* 🚀

---

**License**: This guide is provided as-is for educational purposes. Always test changes in a non-production environment first.  
**Feedback**: Found an error or have a suggestion? Contribute back to the community!  
**Last Reviewed**: March 2026 | **Next Review Due**: September 2026