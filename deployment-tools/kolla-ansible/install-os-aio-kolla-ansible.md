# 🚀 OpenStack Single-Node Installation using Kolla Ansible  
### *Based on Official Guide: [Kolla Ansible 2025.1 Development Quickstart](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart-development.html)*  
**✅ All-in-One (AIO) Setup | ✅ VirtualBox VM | ✅ Fixed Common Issues**

> 📌 **Your Environment**:  
> - Ubuntu 24.04 / Rocky Linux 9  
> - 3 NICs:  
>   - `enp0s3` → NAT (`10.0.2.15`) – Internet  
>   - `enp0s8` → Host-Only (`192.168.10.10`) – Management  
>   - `enp0s9` → Bridged (`192.168.102.221`) – External/Neutron  
> - VIP: `192.168.10.250`  
> - Horizon Access: `http://192.168.10.10` or `http://192.168.102.221`

---

## 🔧 Prerequisites

Ensure your VM has:
- ✅ 2+ vCPUs
- ✅ 8+ GB RAM
- ✅ 40+ GB Disk
- ✅ 3 Network Interfaces (as described)
- ✅ OS: Ubuntu 24.04 LTS or Rocky Linux 9

Run all commands as **root** or with `sudo`.

---

## Step 1: Update System & Install Dependencies
Update system
```bash
apt update && apt upgrade -y
```
Install required build tools and Python dependencies
```
apt install -y git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev python3-venv
```

> 💡 For **Rocky/CentOS**, use:
>
> ```bash
> dnf install -y git python3-devel libffi-devel gcc openssl-devel python3-libselinux
> ```

---

## Step 2: Create and Activate Virtual Environment
Create virtual environment
```bash
python3 -m venv /opt/venv-kolla
```
Activate it
```
source /opt/venv-kolla/bin/activate
```
Upgrade pip
```
pip install -U pip
```

> ⚠️ Keep this activated for all steps!

---

## Step 3: Install Kolla-Ansible from Git (Development Method)
Clone the stable branch
```bash
git clone --branch stable/2025.1 https://opendev.org/openstack/kolla-ansible
```
Install kolla-ansible and its dependency (kolla)
```
pip install ./kolla-ansible
```

---

## Step 4: Create Kolla Configuration Directory
Create config directory
```bash
sudo mkdir -p /etc/kolla
```
Allow current user to write
```
sudo chown $USER:$USER /etc/kolla
```

---

## Step 5: Copy Configuration Files
Copy globals.yml and passwords.yml
```bash
cp -r kolla-ansible/etc/kolla/* /etc/kolla/
```
Copy inventory files (all-in-one and multinode)
```
cp kolla-ansible/ansible/inventory/* .
```

---

## Step 6: Install Ansible Galaxy Dependencies

```bash
kolla-ansible install-deps
```

---

## Step 7: Generate Random Passwords
Run the password generator script
```bash
cd kolla-ansible/tools
./generate_passwords.py
```

> ✅ This fills `/etc/kolla/passwords.yml` with secure passwords.

---

## Step 8: Configure `globals.yml`

Edit the main config file:

```bash
nano /etc/kolla/globals.yml
```

Set these values:

```yaml
# Base OS for containers
kolla_base_distro: "ubuntu"

# OpenStack release (optional, auto-detected)
# kolla_openstack_release: "2025.1"

# Management network interface (host-only)
network_interface: "enp0s8"

# External Neutron interface (bridged, no IP assigned)
neutron_external_interface: "enp0s9"

# Virtual IP for internal services (must be unused)
kolla_internal_vip_address: "192.168.10.250"

# Enable key services
enable_heat: "yes"
enable_horizon: "yes"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"

# Logging
docker_namespace: "quay.io/kolla"
kolla_install_type: "source"
```

> 💡 Save and exit: `Ctrl+O` → Enter → `Ctrl+X`

---

## Step 9: Bootstrap Server

Installs Docker and other host-level dependencies.
Run bootstrap
```bash
cd kolla-ansible/tools
./kolla-ansible bootstrap-servers -i ../../all-in-one
```

---

## Step 10: Fix Ansible Runtime Missing Modules (CRITICAL!)

This prevents `ModuleNotFoundError: docker`, `dbus`, etc.
Install required packages in Kolla's isolated Ansible runtime
```bash
/opt/ansible-runtime/bin/pip install docker dbus-python
```
# Verify installation
```
/opt/ansible-runtime/bin/python3.12 -c "import docker; print('Docker OK')"
/opt/ansible-runtime/bin/python3.12 -c "import dbus; print('DBus OK')"
```

> ✅ This step avoids prechecks failure.

---

## Step 11: Run Pre-Deployment Checks

```bash
kolla-ansible prechecks -i ./all-in-one
```

> ✅ If any test fails, check logs and fix (e.g., time sync, disk space).

---

## Step 12: Deploy OpenStack

```bash
kolla-ansible deploy -i ./all-in-one
```

> 🕐 Takes 10–20 minutes. Wait for `PLAY RECAP` with no failures.

---

## Step 13: Generate Admin Credentials

Create the required directory
```
sudo mkdir -p /etc/kolla/ansible/inventory
```
Copy your all-in-one inventory file to the expected path
```
cp ./all-in-one /etc/kolla/ansible/inventory/all-in-one
```
> 💡 Make sure you are running this from a directory where ./all-in-one exists (e.g., your home directory). 
Run post-deploy
```bash
cd kolla-ansible/tools
./kolla-ansible post-deploy
```
##  Optional: Always Use Explicit Inventory (Recommended)
Instead of relying on default paths, always specify the inventory with -i:
```
kolla-ansible post-deploy -i ./all-in-one
```
> ✅ Creates:
> - `/etc/kolla/admin-openrc.sh`
> - `/etc/kolla/clouds.yaml`

---

## Step 14: Install OpenStack CLI
Install OpenStack client
```bash
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2025.1
```

---

## Step 15: Configure CLI Access
Create config directory
```bash
mkdir -p ~/.config/openstack

# Copy clouds.yaml
cp /etc/kolla/clouds.yaml ~/.config/openstack/
```

Test login:

```bash
source /etc/kolla/admin-openrc.sh
openstack token issue
```

> ✅ You should see a token — success!

---

## Step 16: Run Demo Script (Optional)

Creates sample network, image, and instance.
Source credentials
```bash
source /etc/kolla/admin-openrc.sh

# Run init script
kolla-ansible/tools/init-runonce
```

> ⚠️ Warning: Only for testing. May conflict with custom setups.

---

## Step 17: Access Horizon Dashboard

Open in browser:

```
http://192.168.10.10
```
or
```
http://192.168.102.221
```

> 🔐 Login with:
> - **Username:** `admin`
> - **Password:** Check `/etc/kolla/passwords.yml` → `keystone_admin_password`

Get password:

```bash
grep keystone_admin_password /etc/kolla/passwords.yml
```

---

## 🛠 Troubleshooting & Common Fixes (MUST READ)

### ❌ Problem: `ModuleNotFoundError: No module named 'docker'`
**Fix:**
```bash
/opt/ansible-runtime/bin/pip install docker
```

### ❌ Problem: `ModuleNotFoundError: No module named 'dbus'`
**Fix:**
```bash
/opt/ansible-runtime/bin/pip install dbus-python
```

### ❌ Problem: `inventory ... does not exist`
**Cause:** Wrong path used.
**Fix:**
```bash
cp ./all-in-one /etc/kolla/ansible/inventory/all-in-one -p
```
Or always use `-i ./all-in-one`.

### ❌ Problem: `prechecks` fail due to NTP/time sync
**Fix:**
```bash
timedatectl set-ntp on
timedatectl status
```

### ❌ Problem: Instances can't access external network
**Cause:** `neutron_external_interface` has an IP.
**Fix:**
```bash
sudo ip addr flush dev enp0s9
```

### ❌ Problem: Horizon not accessible
**Fix:**
- Disable firewall:
  ```bash
  ufw disable
  ```
- Or allow ports:
  ```bash
  ufw allow 80/tcp && ufw allow 443/tcp
  ```

### ❌ Problem: `init-runonce` fails or hangs
**Fix:**
- Ensure `neutron_external_interface` is up but without IP.
- Restart Neutron:
  ```bash
  kolla-ansible stop -i ./all-in-one --tags neutron
  kolla-ansible deploy -i ./all-in-one --tags neutron
  ```

---

## 🧹 Cleanup (Optional)

To completely remove OpenStack:

```bash
kolla-ansible destroy -i ./all-in-one --yes-i-really-really-mean-it
```

> ⚠️ This removes **all data, containers, and configurations**.

---

## 🔄 Useful Commands

| Task | Command |
|------|--------|
| Redeploy all | `kolla-ansible deploy -i ./all-in-one` |
| Redeploy Neutron only | `kolla-ansible deploy -i ./all-in-one --tags neutron` |
| Stop all services | `kolla-ansible stop -i ./all-in-one` |
| View container logs | `sudo docker logs <container_name>` |
| List running containers | `sudo docker ps` |

---

## 📚 Official Resources

- **Main Guide**: [https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart-development.html](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart-development.html)
- **Services Reference**: [https://docs.openstack.org/kolla-ansible/latest/admin/services.html](https://docs.openstack.org/kolla-ansible/latest/admin/services.html)
- **Networking Guide**: [https://docs.openstack.org/kolla-ansible/latest/admin/networking.html](https://docs.openstack.org/kolla-ansible/latest/admin/networking.html)

---

✅ **Congratulations!**  
You now have a fully working **single-node OpenStack cloud** using Kolla Ansible — perfect for learning, testing, or development.

Use Horizon, CLI, or APIs to explore Nova, Neutron, Glance, Cinder, and more!

---

📌 *Created by: [Your Name]*  
📅 *Date: 2025-04-05*  
🏷️ *Tags: OpenStack, Kolla-Ansible, AIO, VirtualBox, Cloud, DevOps*

---

> ❤️ **Special Thanks**: This guide includes fixes based on real deployment issues reported by users. Your persistence made this better!

---
