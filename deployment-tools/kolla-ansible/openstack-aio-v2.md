# 🚀 OpenStack Single-Node Deployment using Kolla Ansible  
### *Based on Official Guide: [Kolla Ansible 2025.1 Development Quickstart](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart-development.html)*  
**✅ All-in-One (AIO) Setup | ✅ VirtualBox VM | ✅ Fixed Common Issues**

> ✅ **Perfect for Evaluation, Learning Development & Production**  
> 🔧 Deploy OpenStack All-in-One with Kolla Ansible — Fast, Reliable & Professional

---

## 📌 Table of Contents
- [About Kolla Ansible](#-about-kolla-ansible)
- [Prerequisites](#-prerequisites)
- [Step-by-Step Installation](#-step-by-step-installation)
- [Verify & Use OpenStack](#-verify--use-openstack)
- [Manage & Maintain Your Cloud](#-manage--maintain-your-cloud)
- [Cleanup / Uninstall](#%EF%B8%8F-cleanup--uninstall)
- [🔧 Troubleshooting & Common Issues](#-troubleshooting--common-issues)
- [📜 License & Credits](#-license--credits)

---

## 💡 About Kolla Ansible

[Kolla Ansible](https://docs.openstack.org/kolla-ansible/) is an official OpenStack project that uses **Ansible** to deploy containerized OpenStack services via **Docker**. It’s ideal for:
- Rapid deployment of OpenStack for testing or evaluation.
- Consistent, repeatable infrastructure as code.
- Production-ready architecture (scalable from single-node to multi-node).

> ✅ **Why Use Kolla Ansible?**
> - Fully automated setup
> - Built-in high availability (VIP + keepalived)
> - Easy upgrades and maintenance
> - Supports Ubuntu, Rocky Linux, CentOS

This guide uses the **development-style installation method** (cloning from Git) as per the [official Kolla Ansible quickstart](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart-development.html).

---

## ⚙️ Prerequisites

Ensure your VM meets these requirements:

| Requirement | Details |
|-----------|--------|
| **OS** | Ubuntu 24.04 LTS or Rocky Linux 9 |
| **RAM** | 8 GB or more |
| **Disk** | 40+ GB free space |
| **Network Interfaces** | 3 NICs (NAT, Host-Only, Bridged) |
| **User Privileges** | Root access or sudo rights |

### 🖥️  **Our Environment**: 

> ✅ 2+ vCPUs

> ✅ 8+ GB RAM

> ✅ 40+ GB Disk

> ✅ OS: Ubuntu 24.04.X LTS

> ✅ 3 Network Interfaces (as described)

| Interface | Type | IP Address | Purpose |
|---------|------|------------|--------|
| `enp0s3` | NAT | `10.0.2.15` | Internet access |
| `enp0s8` | Host-Only | `192.168.10.10` | Management network |
| `enp0s9` | Bridged | `192.168.102.221` | External network for instances |

We'll use:
- `enp0s8` → `network_interface`
- `enp0s9` → `neutron_external_interface`
- VIP → `192.168.10.250`

Run all commands as **root** or with `sudo`.

---

## ➕ Step-by-Step Installation

> 💡 Run all commands as **root** or use `sudo`.

### 1. Update System & Install Dependencies

#### Update package list
```
sudo apt update && sudo apt upgrade -y
```
#### Install system build tools and Python dependencies
```
apt install -y git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev python3-venv
```

> 🐧 For **Rocky/CentOS**, use:
>
> ```bash
> dnf install -y git python3-devel libffi-devel gcc openssl-devel python3-libselinux
> ```

---

### 2. Create Virtual Environment

```bash
python3 -m venv /opt/venv-kolla
source /opt/venv-kolla/bin/activate
pip install --upgrade pip
pip install docker
pip install dbus-python
```

> 🔁 Keep this environment activated for all steps!

---

### 3. Clone & Install Kolla-Ansible from Git

```bash
git clone --branch stable/2025.1 https://opendev.org/openstack/kolla-ansible
```
#### Install Python dependencies
```
pip install ./kolla-ansible
```

---

### 4. Create Config Directory

```bash
mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla
```

---

### 5. Copy Configuration Files

```bash
cp -r kolla-ansible/etc/kolla/* /etc/kolla/
cp kolla-ansible/ansible/inventory/* .
```

---

### 6. Install Ansible Galaxy Dependencies

```bash
kolla-ansible install-deps
```

---

### 7. Generate Passwords

```bash
kolla-ansible/tools/generate_passwords.py
```

> 🔐 All passwords are now securely generated in `/etc/kolla/passwords.yml`

---

### 8. Configure `globals.yml`

Edit the main config file:

```bash
nano /etc/kolla/globals.yml
```

Update with your settings:

```yaml
# Base OS for containers
kolla_base_distro: "ubuntu"

# OpenStack release
kolla_install_type: "source"

# Management interface (host-only)
network_interface: "enp0s8"

# External interface for Neutron (bridged, no IP!)
neutron_external_interface: "enp0s9"

# Virtual IP for internal traffic (must be unused)
kolla_internal_vip_address: "192.168.10.250"

# Enable key services
enable_haproxy: "yes"
enable_keystone: "yes"
enable_glance: "yes"
enable_nova: "yes"
enable_neutron: "yes"
enable_placement: "yes"
enable_horizon: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
enable_heat: "yes"
```

Save and exit (`Ctrl+O`, Enter, `Ctrl+X`).

---

### 9. Bootstrap the Server

Installs Docker and other required tools.

```bash
kolla-ansible bootstrap-servers -i all-in-one
```

---

### 10. Run Pre-Deployment Checks

```bash
kolla-ansible prechecks -i all-in-one
```

> ❌ If it fails due to missing Python modules, see **[Troubleshooting](#-troubleshooting--common-issues)** below.

---

### 11. Deploy OpenStack

Deploy all services in containers:

```bash
kolla-ansible deploy -i all-in-one
```

> 🕐 This may take 10–20 minutes.

✅ Wait for: `PLAY RECAP` with **0 failed tasks**.

---

### 12. Generate Admin Credentials

```bash
kolla-ansible post-deploy -i ~/all-in-one
```

> ✅ Output: `/etc/kolla/clouds.yaml` and `/etc/kolla/admin-openrc.sh`

---

### 13. Install OpenStack CLI

```bash
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2025.1
```

---

### 14. Source Admin Environment

```bash
source /etc/kolla/admin-openrc.sh
```

Test connection:

```bash
openstack token issue
```

> ✅ You should see a valid authentication token.

---

### 15. (Optional) Run Demo Script

Creates sample network, image, and instance.

```bash
kolla-ansible/tools/init-runonce
```

> ⚠️ Only for testing! Do not run in production.

---

## 🔍 Verify & Use OpenStack

### Access Horizon Dashboard

Open in browser:
```
http://192.168.10.10
or
http://192.168.102.221
```

Login with:
- **Username:** `admin`
- **Password:** See `/etc/kolla/passwords.yml` → `keystone_admin_password`

Get password:
```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```

---

## 🛠 Manage & Maintain Your Cloud

### Useful Commands

| Task | Command |
|------|--------|
| Redeploy (e.g., after config change) | `kolla-ansible deploy -i ~/all-in-one` |
| Restart specific service (Nova) | `kolla-ansible deploy -i ~/all-in-one --tags nova` |
| Stop all containers | `kolla-ansible stop -i ~/all-in-one` |
| Pull latest images | `kolla-ansible pull -i ~/all-in-one` |
| View logs | `docker logs <container_name>` |
| List running containers | `docker ps` |

---

### Upgrade OpenStack (Future)

When upgrading to a new version:

```bash
# Pull new images
kolla-ansible pull -i ~/all-in-one

# Run upgrade playbook
kolla-ansible upgrade -i ~/all-in-one
```

> ⚠️ Always backup first!

---

## 🧹 Cleanup / Uninstall

To completely remove OpenStack:

```bash
kolla-ansible destroy -i ~/all-in-one --yes-i-really-really-mean-it
```

> ⚠️ **Warning**: This deletes **all data**, including volumes and databases.

Then remove config and virtual env:

```bash
rm -rf /etc/kolla
rm -rf /opt/venv-kolla
rm -rf ~/kolla-ansible
```

---

## 🔧 Troubleshooting & Common Issues

Here are **real problems you faced** and their **perfect fixes**, plus other likely issues.

---

### ❌ Issue 1: `ModuleNotFoundError: No module named 'docker'`

**Error during `prechecks`:**
```
failed: [localhost] ... ModuleNotFoundError: No module named 'docker'
```

**✅ Solution:**

Install `docker` in the **Ansible runtime environment**:

```
pip install docker
```
or
```bash
/opt/ansible-runtime/bin/pip install docker
```

---

### ❌ Issue 2: `ModuleNotFoundError: No module named 'dbus'`

**Error:**
```
import dbus; ModuleNotFoundError: No module named 'dbus'
```

**✅ Solution:**

Install `dbus-python` in the Ansible runtime:

```
pip install dbus-python
```
or

```bash
/opt/ansible-runtime/bin/pip install dbus-python
```

> 🔗 Requires system package: `libdbus-glib-1-dev` (already installed above).

---

### ❌ Issue 3: `Inventory path does not exist` during `post-deploy`

**Error:**
```
Kolla inventory /etc/kolla/ansible/inventory/all-in-one is invalid: Path does not exist
```

**✅ Solution:**

Create the correct directory and copy the file:

```bash
mkdir -p /etc/kolla/ansible/inventory
cp all-in-one /etc/kolla/ansible/inventory/all-in-one
```

Or always specify `-i`:

```bash
kolla-ansible post-deploy -i all-in-one
```

---

### ❌ Issue 4: Neutron external network not working

**Symptoms:**
- Instances can't reach internet
- Floating IPs not accessible

**✅ Fix:**

Ensure `neutron_external_interface` has **no IP address**:

```bash
ip addr flush dev enp0s9
```

And disable any network manager control over it.

---

### ❌ Issue 5: Time synchronization failure

**Error in prechecks about NTP.**

**✅ Fix:**

Enable time sync:

```bash
timedatectl set-ntp true
timedatectl status
```

---

### ❌ Issue 6: Not enough memory or disk space

**Symptom:** Docker fails to start or containers crash.

**✅ Fix:**
- Allocate **at least 8GB RAM** and **40GB disk** to VM.
- Monitor usage: `df -h`, `free -h`

---

### ❌ Issue 7: Fixing "Volume group cinder-volumes not found" Error

#### 1. Error Overview
**Error Message:**
```text
fatal: [localhost]: FAILED! => {"changed": false, "cmd": ["vgs", "cinder-volumes"], ... 
"stderr": "Volume group \"cinder-volumes\" not found"}
```
**When it happens:** 
During `kolla-ansible prechecks` or `deploy`.

#### 2. Root Cause
*   **Default Behavior:** OpenStack Cinder uses LVM storage by default.
*   **Requirement:** It looks for a specific Volume Group named `cinder-volumes`.
*   **Your Setup:** You have only one disk (used for OS). No extra disk exists for Cinder.
*   **Result:** Precheck fails because the storage group is missing.

#### 3. Solution: Create a Loopback Device
We will create a large file on your existing disk and treat it as a virtual hard disk for Cinder.

#### Step 1: Create a Virtual Disk File
Create a 20GB file named `cinder-volumes.img` in the root directory.
```bash
dd if=/dev/zero of=/cinder-volumes.img bs=1M count=20480
```

### Step 2: Setup Loop Device
Connect the file to a loop device (e.g., `/dev/loop0`).
```bash
losetup /dev/loop0 /cinder-volumes.img
```

### Step 3: Create Physical Volume (PV)
Initialize the loop device for LVM.
```bash
pvcreate /dev/loop0
```

### Step 4: Create Volume Group (VG)
Create the required group named `cinder-volumes`.
```bash
vgcreate cinder-volumes /dev/loop0
```

#### 4. Verification
**Check if VG exists:**
```bash
vgs
```
*You should see `cinder-volumes` in the list.*

**Re-run Prechecks:**
```bash
kolla-ansible prechecks -i all-in-one
```
*The Cinder error should be gone.*

## 5. Important Notes

#### A. Persistence (After Reboot)
The loop device will disconnect after reboot. To fix this automatically:
1.  Open `/etc/rc.local` (create if not exists).
2.  Add these lines before `exit 0`:
    ```bash
    losetup /dev/loop0 /cinder-volumes.img
    vgchange -ay
    ```
3.  Make sure `/etc/rc.local` is executable.

#### B. Storage Capacity
*   **Total Size:** 20 GB.
*   **Usable Size:** ~18 GB (some space used for LVM metadata).
*   **Usage:** You can create multiple volumes (e.g., 10 volumes of 1GB each). Once full, you cannot create new volumes.

#### C. Performance
*   **Lab/Testing:** Perfectly fine.
*   **Production:** Not recommended. It is slower than a physical disk.

#### D. Future Migration to Ceph
If you add Ceph storage later:
1.  Update `globals.yml` to enable `cinder_backend_ceph`.
2.  Disable `cinder_backend_lvm`.
3.  Re-run `kolla-ansible deploy`.
4.  You do not need to delete the loopback file, but Cinder will stop using it for new volumes.

---

## 📜 License & Credits

- **Guide Version:** 2025.1
- **Author:** Sumon Paul
- **Source:** Based on [OpenStack Kolla Ansible Docs](https://docs.openstack.org/kolla-ansible/2025.1/)
- **License:** MIT (Feel free to reuse and modify)

---

✅ **Congratulations!**  
You’ve successfully deployed a fully functional **single-node OpenStack cloud** using **Kolla Ansible**.

Use this environment to learn Nova, Neutron, Cinder, Horizon, and more!

---

📬 **Need Help?**  
Have questions or improvements? Open an issue or contact me!

✨ **Happy Cloud Building!**

--- 
