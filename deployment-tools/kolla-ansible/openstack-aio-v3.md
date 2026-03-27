# 🚀 OpenStack Single-Node Installation & Complete Operations Guide  
### *Using Kolla Ansible 2025.1 | All-in-One (AIO) | VirtualBox Ready*  
**✅ Official Development Quickstart Based | ✅ Production-Grade Practical Guide | ✅ Copy-Paste Commands**

> 📌 **Your Environment Reference**:  
> - **OS**: Ubuntu 24.04 LTS / Rocky Linux 9  
> - **VM**: VirtualBox with 3 NICs  
>   - `enp0s3` → NAT (`10.0.2.15`) – Internet Gateway  
>   - `enp0s8` → Host-Only (`192.168.10.10`) – Management Network  
>   - `enp0s9` → Bridged (`192.168.102.221`) – External/Provider Network  
> - **VIP**: `192.168.10.250` (Internal API endpoint)  
> - **Horizon**: `http://192.168.10.10` or `http://192.168.102.221`  
> - **Guide Version**: 2025.1 (Zed/2025.1 Release)

---

## 📑 Table of Contents

1. [🎯 Introduction & What You'll Learn](#-introduction--what-youll-learn)  
2. [🔧 Prerequisites & System Requirements](#-prerequisites--system-requirements)  
3. [📦 Step 1-2: System Update & Virtual Environment](#-step-1-2-system-update--virtual-environment)  
4. [🔗 Step 3: Install Kolla-Ansible (Official Method + Version Check)](#-step-3-install-kolla-ansible-official-method--version-check)  
5. [⚙️ Step 4-8: Configuration Setup](#️-step-4-8-configuration-setup)  
6. [🚀 Step 9-12: Bootstrap & Deployment](#-step-9-12-bootstrap--deployment)  
7. [🔐 Step 13-17: CLI Setup & Horizon Access](#-step-13-17-cli-setup--horizon-access)  
8. [✅ Step 18: Verify OpenStack Installation](#-step-18-verify-openstack-installation)  
9. [🌐 Step 19: Production-Grade Network Configuration](#-step-19-production-grade-network-configuration)  
10. [💻 Step 20: Flavor Management – Compute Resources](#-step-20-flavor-management--compute-resources)  
11. [💾 Step 21: Volume & Storage Management (Cinder)](#-step-21-volume--storage-management-cinder)  
12. [🔑 Step 22: Keypairs & SSH Access Setup](#-step-22-keypairs--ssh-access-setup)  
13. [🖥️ Step 23: Create & Launch Your First Instance](#️-step-23-create--launch-your-first-instance)  
14. [🔄 Step 24: Instance Operations – Start, Stop, Snapshot, Resize](#-step-24-instance-operations--start-stop-snapshot-resize)  
15. [🛡️ Step 25: Security Groups & Firewall Rules](#️-step-25-security-groups--firewall-rules)  
16. [📊 Step 26: Monitoring & Daily Maintenance](#-step-26-monitoring--daily-maintenance)  
17. [🧰 Troubleshooting – Common Issues & Fixes](#-troubleshooting--common-issues--fixes)  
18. [✨ Best Practices for Production Readiness](#-best-practices-for-production-readiness)  
19. [📚 Official Resources & Further Learning](#-official-resources--further-learning)  

---

## 🎯 Introduction & What You'll Learn

Welcome to your **complete OpenStack journey**! 🎉

This guide is not just about installation — it's a **story of building, configuring, and operating** a real OpenStack cloud on your laptop using VirtualBox.

> 🧭 **Learning Path**:  
> 1. Install OpenStack using Kolla Ansible (containerized, production-style)  
> 2. Verify every service is running correctly  
> 3. Configure networking so your VMs can reach the internet  
> 4. Create flavors, volumes, keypairs, and security rules  
> 5. Launch your first instance and connect via SSH  
> 6. Manage, monitor, and maintain your cloud like a pro  

All commands are **copy-paste ready**, explained in simple English, with real-world examples.

Let's begin! 👇

---

## 🔧 Prerequisites & System Requirements

Before we start, ensure your VM meets these **minimum requirements**:

| Resource | Requirement | Why? |
|----------|-------------|------|
| **vCPUs** | 2+ cores | OpenStack services need CPU for orchestration |
| **RAM** | 8+ GB | Containers + services consume memory |
| **Disk** | 40+ GB | Images, volumes, logs need space |
| **NICs** | 3 interfaces | Management, external, and internet access |
| **OS** | Ubuntu 24.04 / Rocky 9 | Officially supported by Kolla Ansible |

> 💡 **Pro Tip**: Enable nested virtualization in VirtualBox if you plan to launch VMs inside OpenStack:
> ```bash
> VBoxManage modifyvm "YourVMName" --nested-hw-virt on
> ```

---

## 📦 Step 1-2: System Update & Virtual Environment

### Update System Packages

```bash
# Update package index and upgrade existing packages
sudo apt update && sudo apt upgrade -y
```

> ✅ This ensures you have the latest security patches and dependencies.

### Install Build Dependencies

```bash
# Install Python build tools and libraries
sudo apt install -y git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev python3-venv
```

> 💡 For **Rocky/CentOS**, use:
> ```bash
> sudo dnf install -y git python3-devel libffi-devel gcc openssl-devel python3-libselinux
> ```

### Create & Activate Virtual Environment

```bash
# Create isolated Python environment
python3 -m venv /opt/venv-kolla

# Activate the environment
source /opt/venv-kolla/bin/activate

# Upgrade pip inside venv
pip install -U pip
```

> ⚠️ **Important**: Keep this terminal session active. All `pip` and `kolla-ansible` commands must run inside this venv.

---

## 🔗 Step 3: Install Kolla-Ansible (Official Method + Version Check)

### 🌐 Official Repository Reference

Kolla Ansible source code is hosted on **OpenDev** (OpenStack's official code hosting):

🔗 **Repository**: https://opendev.org/openstack/kolla-ansible  
🔗 **Branches**: `stable/2025.1`, `stable/zed`, `master`  
🔗 **Documentation**: https://docs.openstack.org/kolla-ansible/2025.1/

### 🔍 How to Check Latest Available Version

Before cloning, verify the latest stable release:

```bash
# Check available branches in the official repo
git ls-remote --heads https://opendev.org/openstack/kolla-ansible | grep stable

# Example output:
# refs/heads/stable/2025.1
# refs/heads/stable/zed
# refs/heads/stable/yoga
```

> ✅ **Recommendation**: Use `stable/2025.1` for the latest long-term supported release.

### 📥 Clone & Install Kolla-Ansible

```bash
# Clone the stable branch (development method)
git clone --branch stable/2025.1 https://opendev.org/openstack/kolla-ansible

# Install kolla-ansible and its dependency (kolla) from local path
pip install ./kolla-ansible
```

> 🎯 Why local install?  
> The development guide recommends installing from cloned source to ensure you have the exact version matching the documentation, with access to tools like `generate_passwords.py` and `init-runonce`.

### 🗂️ Create Configuration Directory

```bash
# Create /etc/kolla for configs
sudo mkdir -p /etc/kolla

# Allow your user to write configs
sudo chown $USER:$USER /etc/kolla
```

### 📋 Copy Configuration Files

```bash
# Copy default config files (globals.yml, passwords.yml)
cp -r kolla-ansible/etc/kolla/* /etc/kolla/

# Copy inventory files (all-in-one, multinode) to current directory
cp kolla-ansible/ansible/inventory/* .
```

> ✅ You now have:
> - `/etc/kolla/globals.yml` → Main config file  
> - `/etc/kolla/passwords.yml` → Service passwords  
> - `./all-in-one` → Inventory for single-node deployment  

---

## ⚙️ Step 4-8: Configuration Setup

### Install Ansible Galaxy Dependencies

```bash
# Install required Ansible roles from Galaxy
kolla-ansible install-deps
```

> ✅ This downloads roles like `docker`, `python`, `common` needed for deployment.

### Generate Random Passwords

```bash
# Navigate to tools directory
cd kolla-ansible/tools

# Run password generator
./generate_passwords.py

# Return to project root
cd -
```

> 🔐 This fills `/etc/kolla/passwords.yml` with secure random passwords for all services.

### Configure `globals.yml` – The Heart of Deployment

Edit the main config file:

```bash
nano /etc/kolla/globals.yml
```

Add or update these **critical settings**:

```yaml
# === Base Settings ===
kolla_base_distro: "ubuntu"          # Use "rocky" for Rocky Linux
kolla_install_type: "source"         # Build from source (dev) or use pre-built

# === Network Interfaces (YOUR VM SPECIFIC) ===
network_interface: "enp0s8"          # Host-Only: Management traffic
neutron_external_interface: "enp0s9" # Bridged: External/Provider network (NO IP!)

# === Virtual IP for HA ===
kolla_internal_vip_address: "192.168.10.250"  # Unused IP in management subnet

# === Enable Key Services ===
enable_horizon: "yes"                # Web dashboard
enable_cinder: "yes"                 # Block storage
enable_cinder_backend_lvm: "yes"     # Use LVM for Cinder (simple for AIO)
enable_heat: "yes"                   # Orchestration (optional)

# === Container Registry ===
docker_namespace: "quay.io/kolla"    # Official Kolla images
```

> 💡 **Critical Note for `neutron_external_interface`**:  
> The bridged interface (`enp0s9`) must be **active but without an IP address**. If it has an IP, instances won't get external access.
> ```bash
> # Remove IP if assigned
> sudo ip addr flush dev enp0s9
> ```

Save and exit: `Ctrl+O` → Enter → `Ctrl+X`

---

## 🚀 Step 9-12: Bootstrap & Deployment

### Bootstrap Servers – Install Docker & Dependencies

```bash
# Run bootstrap playbook (from tools directory)
cd kolla-ansible/tools
./kolla-ansible bootstrap-servers -i ../../all-in-one
cd -
```

> ✅ This installs Docker, containerd, and system packages required for Kolla containers.

### ⚠️ Fix Ansible Runtime Missing Modules (CRITICAL!)

Kolla Ansible uses an isolated Ansible runtime that may miss Python modules.

```bash
# Install missing modules in Kolla's Ansible runtime
/opt/ansible-runtime/bin/pip install docker dbus-python

# Verify installation
/opt/ansible-runtime/bin/python3 -c "import docker; print('✅ Docker module OK')"
/opt/ansible-runtime/bin/python3 -c "import dbus; print('✅ DBus module OK')"
```

> ❌ Without this, `prechecks` will fail with `ModuleNotFoundError`.

### Run Pre-Deployment Checks

```bash
# Validate host readiness
kolla-ansible prechecks -i ./all-in-one
```

> ✅ All checks should pass: `ok=XX, failed=0`  
> ❌ If failed: Check NTP sync, disk space, interface names, and fix before proceeding.

### Deploy OpenStack – The Main Event! 🎉

```bash
# Deploy all OpenStack services in containers
kolla-ansible deploy -i ./all-in-one
```

> 🕐 **Expect 15-30 minutes** depending on internet speed and VM performance.  
> ✅ Wait for `PLAY RECAP` with `failed=0`.  
> 🎯 This pulls container images, configures services, and starts OpenStack!

---

## 🔐 Step 13-17: CLI Setup & Horizon Access

### Generate Admin Credentials

```bash
# Run post-deploy to generate clouds.yaml and admin-openrc.sh
cd kolla-ansible/tools
./kolla-ansible post-deploy
cd -
```

> ✅ Creates:
> - `/etc/kolla/admin-openrc.sh` → Environment variables for CLI  
> - `/etc/kolla/clouds.yaml` → Standard OpenStack config file  

### Install OpenStack CLI Client

```bash
# Install official OpenStack client with version constraints
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2025.1
```

### Configure CLI Access

```bash
# Create OpenStack config directory
mkdir -p ~/.config/openstack

# Copy clouds.yaml to standard location
cp /etc/kolla/clouds.yaml ~/.config/openstack/
```

### Test Authentication

```bash
# Source admin credentials
source /etc/kolla/admin-openrc.sh

# Request a token – if successful, you're authenticated!
openstack token issue
```

> ✅ Output shows token, project, user – you're in! 🎉

### Access Horizon Dashboard

Open your browser and visit:

```
http://192.168.10.10
```
or
```
http://192.168.102.221
```

> 🔐 **Login Credentials**:
> - **Username**: `admin`  
> - **Project**: `admin`  
> - **Password**: Find it with:
>   ```bash
>   grep keystone_admin_password /etc/kolla/passwords.yml
>   ```

> 🌟 You now have a working OpenStack web interface!

---

## ✅ Step 18: Verify OpenStack Installation

Before building, let's **verify everything is healthy**.

### Check OpenStack Version & Release

```bash
# Source admin credentials first
source /etc/kolla/admin-openrc.sh

# Check OpenStack release version
openstack --version

# List all enabled services and their status
openstack compute service list
openstack network agent list
openstack volume service list
```

> ✅ All services should show `Status: enabled` and `State: up`

### Check Running Containers (Kolla Style)

```bash
# List all Kolla-managed containers
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check specific service logs (e.g., Nova)
sudo docker logs nova_api
```

> 🎯 Tip: Use `sudo docker stats` to monitor resource usage in real-time.

### Verify API Endpoints

```bash
# List all service endpoints
openstack endpoint list

# Test Nova API directly
curl -s http://192.168.10.250:8774/v2.1 | python3 -m json.tool | head -20
```

> ✅ You should see JSON response from Nova API.

### Quick Health Check Script (Copy-Paste)

```bash
echo "🔍 OpenStack Health Check"
echo "========================"
echo "Release: $(openstack --version 2>&1 | head -1)"
echo "Compute Services: $(openstack compute service list -c 'Binary' -f value | wc -l) running"
echo "Network Agents: $(openstack network agent list -c 'Agent Type' -f value | wc -l) active"
echo "Volume Services: $(openstack volume service list -c 'Binary' -f value | wc -l) enabled"
echo "✅ All core services verified!"
```

---

## 🌐 Step 19: Production-Grade Network Configuration

This is **the most critical step** for making your OpenStack useful.

### 🎯 Goal: Instances Should:
1. Get IP from OpenStack network  
2. Reach the internet (via NAT)  
3. Be accessible from outside (via floating IP)  

### Step 19.1: Create Provider Network (External)

```bash
# Source admin credentials
source /etc/kolla/admin-openrc.sh

# Create external provider network (connected to enp0s9)
openstack network create --share \
  --external \
  --provider-physical-network provider \
  --provider-network-type flat provider

# Create subnet for provider network
openstack subnet create --network provider \
  --allocation-pool start=192.168.102.100,end=192.168.102.200 \
  --dns-nameserver 8.8.8.8 \
  --gateway 192.168.102.1 \
  --subnet-range 192.168.102.0/24 provider-subnet
```

> 💡 Replace `192.168.102.1` with your actual bridge network gateway.

### Step 19.2: Create Private Network (For Instances)

```bash
# Create private network for instances
openstack network create project-net

# Create subnet for private network
openstack subnet create --network project-net \
  --dns-nameserver 8.8.8.8 \
  --subnet-range 10.10.10.0/24 project-subnet

# Create router to connect private to external
openstack router create project-router

# Connect router to private subnet
openstack router add subnet project-router project-subnet

# Set gateway to external network
openstack router set --external-gateway provider project-router
```

> 🎯 Now instances on `project-net` can reach internet via NAT!

### Step 19.3: Allocate Floating IP (For External Access)

```bash
# Create floating IP from provider network
openstack floating ip create provider

# Note the floating IP (e.g., 192.168.102.150)
openstack floating ip list
```

> 🌐 This IP can be attached to any instance for direct external access.

### Step 19.4: Test Network Connectivity

```bash
# Create a test security group allowing ICMP and SSH
openstack security group create test-sg
openstack security group rule create --proto icmp test-sg
openstack security group rule create --proto tcp --dst-port 22 test-sg

# Launch a test instance (we'll create flavor/image next)
# For now, just verify network setup:
openstack network show project-net
openstack router show project-router
```

> ✅ If router shows `external_gateway_info`, your NAT is working!

### 🧪 Real-World Example: Web Server Instance

Imagine you're deploying a web app:

1. Instance gets private IP: `10.10.10.50`  
2. Floating IP assigned: `192.168.102.150`  
3. User accesses `http://192.168.102.150` → Router NATs to `10.10.10.50:80`  
4. Instance reaches internet to fetch updates via `192.168.102.1` gateway  

This is **production networking** in action! 🚀

---

## 💻 Step 20: Flavor Management – Compute Resources

Flavors define VM size (CPU, RAM, Disk).

### List Default Flavors

```bash
openstack flavor list
```

### Create Custom Flavors

```bash
# Small dev instance: 1 vCPU, 2GB RAM, 20GB disk
openstack flavor create --id 1 --vcpus 1 --ram 2048 --disk 20 small-dev

# Medium app instance: 2 vCPU, 4GB RAM, 40GB disk
openstack flavor create --id 2 --vcpus 2 --ram 4096 --disk 40 medium-app

# Large DB instance: 4 vCPU, 8GB RAM, 80GB disk
openstack flavor create --id 3 --vcpus 4 --ram 8192 --disk 80 large-db
```

### Set Flavor Metadata (Optional)

```bash
# Add description to flavor
openstack flavor set small-dev --property description="Development & Testing"
```

### Delete Unused Flavors

```bash
openstack flavor delete small-dev
```

> 💡 **Best Practice**: Create flavors matching your workload patterns early.

---

## 💾 Step 21: Volume & Storage Management (Cinder)

Block storage for persistent data.

### Verify Cinder Service

```bash
openstack volume service list
# Should show cinder-volume service as 'up'
```

### Create a Volume

```bash
# Create 10GB volume
openstack volume create --size 10 my-data-volume

# Check volume status
openstack volume show my-data-volume
```

### Attach Volume to Instance

```bash
# After launching instance (see Step 23), attach volume:
openstack server add volume <instance-id> my-data-volume

# Inside instance, format and mount:
# sudo mkfs.ext4 /dev/vdb
# sudo mkdir /data && sudo mount /dev/vdb /data
```

### Create Volume from Snapshot

```bash
# Snapshot existing volume
openstack volume snapshot create --volume my-data-volume snap-001

# Create new volume from snapshot
openstack volume create --snapshot snap-001 restored-volume
```

> 🎯 Use case: Backup production DB volume, restore to test environment.

---

## 🔑 Step 22: Keypairs & SSH Access Setup

Secure SSH access to instances.

### Generate SSH Key Pair (Local Machine)

```bash
# On your laptop (not VM), generate key
ssh-keygen -t ed25519 -f ~/.ssh/openstack-key -C "openstack-access"

# Set secure permissions
chmod 600 ~/.ssh/openstack-key
```

### Import Public Key to OpenStack

```bash
# Source admin credentials
source /etc/kolla/admin-openrc.sh

# Add public key to OpenStack
openstack keypair create --public-key ~/.ssh/openstack-key.pub my-key
```

### Verify Keypair

```bash
openstack keypair list
openstack keypair show my-key
```

> 🔐 This key will be injected into instances at launch for passwordless SSH.

---

## 🖥️ Step 23: Create & Launch Your First Instance

### Download a Test Image (CirrOS)

```bash
# Download lightweight test image
wget https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

# Upload to Glance
openstack image create "cirros-test" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 --container-format bare --public
```

### Launch Instance

```bash
# Launch with all components
openstack server create \
  --image cirros-test \
  --flavor small-dev \
  --network project-net \
  --key-name my-key \
  --security-group test-sg \
  my-first-instance
```

### Monitor Boot Process

```bash
# Watch instance status
openstack server list
openstack server show my-first-instance

# Check console log if stuck
openstack console log show my-first-instance
```

### Assign Floating IP

```bash
# Get floating IP (from Step 19.3)
FLOATING_IP=$(openstack floating ip list -c 'Floating IP Address' -f value | head -1)

# Attach to instance
openstack server add floating ip my-first-instance $FLOATING_IP

# Verify
openstack server show my-first-instance -c addresses -f value
```

### SSH into Instance

```bash
# Connect via SSH (from your laptop)
ssh -i ~/.ssh/openstack-key cirros@$FLOATING_IP

# Test internet access inside instance
ping -c 3 8.8.8.8
curl -s https://ifconfig.me
```

> 🎉 Congratulations! Your instance is running with internet access!

---

## 🔄 Step 24: Instance Operations – Start, Stop, Snapshot, Resize

### Basic Lifecycle Commands

```bash
# Stop instance (graceful shutdown)
openstack server stop my-first-instance

# Start instance
openstack server start my-first-instance

# Reboot instance
openstack server reboot my-first-instance

# Delete instance (use with caution!)
openstack server delete my-first-instance
```

### Create Snapshot (Image Backup)

```bash
# Create snapshot of running instance
openstack server image create --name my-instance-snap my-first-instance

# Verify new image
openstack image list | grep my-instance-snap
```

### Resize Instance (Change Flavor)

```bash
# Resize to larger flavor
openstack server resize --flavor medium-app my-first-instance

# Confirm resize (after verification)
openstack server confirm-resize my-first-instance

# Or revert if issues
openstack server revert-resize my-first-instance
```

> ⚠️ Resize requires instance to support the new flavor's resources.

### Rescue Mode (Recovery)

```bash
# Enter rescue mode (boots from base image, attaches original disk as /dev/vdb)
openstack server rescue my-first-instance

# After fixing, unrescue
openstack server unrescue my-first-instance
```

> 🛠️ Use rescue mode to fix broken instances without data loss.

---

## 🛡️ Step 25: Security Groups & Firewall Rules

Control traffic to/from instances.

### Create Security Group

```bash
openstack security group create web-sg --description "Web Server Rules"
```

### Add Rules

```bash
# Allow HTTP (port 80)
openstack security group rule create --proto tcp --dst-port 80 web-sg

# Allow HTTPS (port 443)
openstack security group rule create --proto tcp --dst-port 443 web-sg

# Allow SSH from specific IP only (more secure)
openstack security group rule create --proto tcp --dst-port 22 --remote-ip 192.168.102.0/24 web-sg

# Allow outbound all traffic (default, but explicit is better)
openstack security group rule create --egress web-sg
```

### Apply to Instance

```bash
# Add security group to existing instance
openstack server add security group my-first-instance web-sg

# Or specify at launch (see Step 23)
```

### List & Manage Rules

```bash
openstack security group rule list web-sg
openstack security group rule delete <rule-id>
```

> 🔐 **Best Practice**: Always restrict SSH to known IPs, not `0.0.0.0/0`.

---

## 📊 Step 26: Monitoring & Daily Maintenance

### Check Service Health Daily

```bash
# One-liner health check
source /etc/kolla/admin-openrc.sh
echo "=== OpenStack Daily Health ==="
openstack compute service list -c 'Host' -c 'Binary' -c 'Status' -c 'State' -f table
openstack network agent list -c 'Host' -c 'Agent Type' -c 'Alive' -f table
```

### Monitor Resource Usage

```bash
# List all instances with flavors
openstack server list --long -c Name -c Flavor -c Status -c Addresses

# Check volume usage
openstack volume list --long -c Name -c Size -c Status

# Monitor container resource usage (Kolla)
sudo docker stats --no-stream
```

### Backup Critical Configs

```bash
# Backup Kolla configs
tar -czf ~/kolla-backup-$(date +%F).tar.gz /etc/kolla

# Backup passwords (store securely!)
cp /etc/kolla/passwords.yml ~/passwords-backup-$(date +%F).yml
chmod 600 ~/passwords-backup-*.yml
```

### Update OpenStack Services (Minor)

```bash
# Re-deploy with updated containers (if using same branch)
kolla-ansible deploy -i ./all-in-one --tags common,config

# Always test in staging first!
```

> 🔄 For major upgrades, follow official upgrade guides – never skip versions.

---

## 🧰 Troubleshooting – Common Issues & Fixes

### ❌ Issue: Instance Has No Internet Access
**Symptoms**: `ping 8.8.8.8` fails inside instance  
**Fix**:
```bash
# 1. Verify router has external gateway
openstack router show project-router -c external_gateway_info

# 2. Ensure external interface has no IP
ip addr show enp0s9 | grep inet

# 3. Check security group allows outbound
openstack security group rule list --egress

# 4. Verify NAT in router namespace (advanced)
sudo ip netns list | grep qrouter
sudo ip netns exec qrouter-<id> iptables -t nat -L POSTROUTING
```

### ❌ Issue: Horizon Shows "Unable to Retrieve" Errors
**Fix**:
```bash
# Check Horizon container logs
sudo docker logs horizon

# Restart Horizon service
kolla-ansible stop -i ./all-in-one --tags horizon
kolla-ansible deploy -i ./all-in-one --tags horizon

# Clear browser cache or try incognito mode
```

### ❌ Issue: `openstack` CLI Commands Hang or Timeout
**Fix**:
```bash
# Verify credentials are sourced
env | grep OS_

# Test API endpoint directly
curl -s http://192.168.10.250:5000/v3

# Check if keepalived is running (for VIP)
sudo docker ps | grep keepalived
```

### ❌ Issue: Cinder Volume Stuck in "creating" State
**Fix**:
```bash
# Check cinder-volume service
openstack volume service list | grep cinder-volume

# Restart cinder services
kolla-ansible stop -i ./all-in-one --tags cinder
kolla-ansible deploy -i ./all-in-one --tags cinder

# Check LVM setup (if using LVM backend)
sudo vgs && sudo lvs
```

### ❌ Issue: SSH Connection Refused to Instance
**Fix**:
```bash
# 1. Verify floating IP is attached
openstack server show my-instance -c addresses

# 2. Check security group allows port 22 from your IP
openstack security group rule list web-sg | grep 22

# 3. Verify keypair was injected (check instance console)
openstack console log show my-instance | grep -i "authorized"

# 4. Test with verbose SSH
ssh -v -i ~/.ssh/openstack-key cirros@<floating-ip>
```

> 🎯 **Golden Rule**: Always check logs – `sudo docker logs <service>` is your best friend.

---

## ✨ Best Practices for Production Readiness

### 🔐 Security
- Disable root login in instances: `sudo sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config`
- Use security groups, not just firewall on instance
- Rotate passwords in `/etc/kolla/passwords.yml` periodically
- Enable TLS for Horizon: `enable_horizon_https: "yes"`

### 📈 Performance
- Monitor disk I/O: `sudo iostat -x 2`
- Set appropriate Cinder LVM filter: `cinder_lvm_filter: "/dev/sd.*/"`
- Use `enable_haproxy: "yes"` for load balancing APIs

### 🔄 Reliability
- Backup `/etc/kolla` and `/var/lib/kolla` regularly
- Test disaster recovery: destroy and redeploy from backup
- Use `kolla-ansible prechecks` before any major change

### 🧹 Maintenance
- Clean old images: `openstack image list --status active -c ID -f value | xargs -I {} openstack image delete {}` (use carefully!)
- Remove unused floating IPs: `openstack floating ip list -c ID -f value | xargs -I {} openstack floating ip delete {}`
- Prune Docker images: `sudo docker image prune -f`

### 📚 Documentation
- Keep this README updated with your customizations
- Document network topology: draw a diagram!
- Log all changes in a change management system

---

## 📚 Official Resources & Further Learning

### 🔗 Official Documentation
- **Kolla Ansible Guide**: https://docs.openstack.org/kolla-ansible/2025.1/
- **OpenStack CLI Reference**: https://docs.openstack.org/python-openstackclient/latest/
- **Networking Guide**: https://docs.openstack.org/neutron/latest/admin/
- **Security Guide**: https://docs.openstack.org/security-guide/

### 🧪 Testing & Validation
- **Tempest**: OpenStack integration test suite
  ```bash
  # Install tempest
  pip install tempest
  # Run basic tests
  tempest run --regex test_list_servers
  ```

### 🌍 Community & Support
- **Ask OpenStack**: https://ask.openstack.org/
- **IRC/Matrix**: #openstack-kolla on libera.chat
- **GitHub Issues**: https://github.com/openstack/kolla-ansible/issues

### 🎓 Next Learning Steps
1. **Multi-node deployment**: Scale beyond single VM
2. **Ceph integration**: Replace LVM with distributed storage
3. **Octavia**: Add load balancer as a service
4. **Zaqar/Mistral**: Explore messaging and workflow services

---

## 🏁 Conclusion & Next Steps

🎉 **You've built a fully functional OpenStack cloud!**

### ✅ What You Can Do Now:
- Launch VMs with custom flavors  
- Attach persistent volumes  
- Configure complex networks with routers & floating IPs  
- Manage users, projects, and quotas  
- Monitor and maintain your cloud  

### 🚀 What's Next?
1. **Automate**: Write Ansible playbooks to deploy apps on OpenStack  
2. **Integrate**: Connect OpenStack to your CI/CD pipeline  
3. **Scale**: Add more compute nodes to your cluster  
4. **Secure**: Implement RBAC, audit logging, and encryption  

### 💬 Final Thought
> "OpenStack is not just software – it's a cloud operating system. You're now the administrator of your own data center in a box."

---

📌 **Created by**: [Your Name]  
📅 **Last Updated**: 2025-04-05  
🏷️ **Tags**: `OpenStack`, `Kolla-Ansible`, `All-in-One`, `VirtualBox`, `DevOps`, `Cloud`, `Tutorial`, `Production-Guide`  

> ❤️ **Acknowledgment**: This guide incorporates real-world fixes from community deployments. Your feedback makes it better!

---

> 🔄 **Update This Guide**:  
> As you customize your OpenStack, add your own sections:  
> - `## 🏢 My Company's Custom Configs`  
> - `## 🛠️ Our Backup Strategy`  
> - `## 📞 Support Contacts`  

**Happy Cloud Building!** ☁️✨