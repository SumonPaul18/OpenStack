# OpenStack Instance Auto-Start Guide

## Table of Contents
1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Solution 1: Enable Nova Service Auto-Start (Recommended)](#3-solution-1-enable-nova-service-auto-start-recommended)
4. [Solution 2: Enable Libvirt Autostart Flag (Crucial)](#4-solution-2-enable-libvirt-autostart-flag-crucial)
5. [Solution 3: Manual Configuration (Fastest Workaround)](#5-solution-3-manual-configuration-fastest-workaround)
6. [Testing the Setup](#6-testing-the-setup)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Introduction
When you reboot your single-node OpenStack server (installed with Kolla-Ansible), the underlying virtual machines (instances) do not start automatically by default. This happens because the `nova-compute` service needs to be configured to resume instances, and the instances themselves must have the correct boot policy set.

This guide provides three methods to solve this issue, focusing on **Solution 1** as the most reliable method for production-like environments.

> **Note:** All commands should be run on your **Controller/Compute node** where Kolla-Ansible is deployed, usually inside the specific Docker containers or on the host depending on the step.

---

## 2. Prerequisites
Before starting, ensure you have:
- Access to the server via SSH as `root` or a sudo user.
- Kolla-Ansible environment variables loaded.
- Basic knowledge of Linux command line.
- Internet access (to install `libvirt-clients` if missing).

Load your OpenStack admin credentials:
```bash
source /etc/kolla/admin-openrc.sh
```

---

## 3. Solution 1: Enable Nova Service Auto-Start (Recommended)
This is the most effective method. We will configure the `nova-compute` service to automatically resume instances that were running before the shutdown.

### Step 3.1: Edit Kolla Ansible Configuration
We need to change the configuration in your `globals.yml` file to tell Nova to resume instances.

1. Open your configuration file (usually located in `/etc/kolla/`):
   ```bash
   nano /etc/kolla/globals.yml
   ```

2. Add or modify the following line under the Nova section:
   ```yaml
   nova_compute_resume_instances: true
   ```
   *If the file doesn't have this section, you can add it at the end.*
### OR

```
nova_enable_resume_guests_state_on_host_boot: true
```

### Step 3.2: Re-deploy Nova Compute
Apply the new configuration using Kolla-Ansible. This will restart the Nova compute container with the new setting.

```bash
kolla-ansible reconfigure -i all-in-one --tags nova
```
*(Replace `<YOUR_INVENTORY_FILE>` with your actual inventory file, e.g., `all-in-one` or `multinode`)*

### Step 3.3: Verify the Setting Inside the Container
To confirm the setting is active, enter the Nova compute container:

```bash
docker exec -it nova_compute bash
```

Inside the container, check the configuration:
```bash
grep "resume_instances" /etc/nova/nova.conf
```
**Expected Output:** `resume_instances = true`

Exit the container:
```bash
exit
```

### OR

```
docker exec -it nova_compute grep "resume_guests_state_on_host_boot" /etc/nova/nova.conf
```
> **Expected Output:** `resume_guests_state_on_host_boot = true`
---

## 4. Solution 2: Enable Libvirt Autostart Flag (Crucial)
**Important:** Only configuring Nova is often not enough. You must explicitly tell **Libvirt** (the hypervisor) to start this specific VM when the host boots.

### Step 4.1: Install Libvirt Clients (If Missing)
The `virsh` command is required to manage Libvirt. If it is not installed on your host, install it now:

```bash
apt update
apt install libvirt-clients -y
```

### Step 4.2: Get Instance UUID
First, identify the unique ID of the instance you want to auto-start.

1. List your instances:
   ```bash
   openstack server list
   ```
2. Copy the **ID** of your target instance (e.g., `a1b2c3d4-e5f6-...`).

### Step 4.3: Set Autostart in Libvirt
You need to run the `virsh autostart` command. In Kolla deployments, `libvirtd` usually runs on the **Host**, but sometimes it is isolated inside the `nova_compute` container.

**Option A: Run on Host (Most Common)**
Try running this directly on your host terminal:
```bash
virsh autostart <INSTANCE_UUID>
```
*(Replace `<INSTANCE_UUID>` with your actual ID)*

**Expected Success Message:**
```text
Domain a1b2c3d4-e5f6-... marked as autostarted
```

**Option B: Run Inside Container (If Option A Fails)**
If you get a connection error on the host, Libvirt is likely inside the Docker container. Run this instead:
```bash
docker exec -it nova_compute virsh autostart <INSTANCE_UUID>
```

### Or Another Way

#### Enable Auto-Start Flag
Use the `openstack server set` command with the `--property` flag to enable auto-start. Replace `<instance-name-or-id>` with your actual instance details.

```bash
openstack server set --property auto_start=True <instance-name-or-id>
```
*Example:*
```bash
openstack server set --property auto_start=True my-vm
```

> **Note:** If you have multiple instances, repeat this command for each one or use a loop script.


### Step 4.4: Verify Configuration - Autostart Status
Confirm that the setting has been saved.

**If using Host:**
```bash
virsh list --all --autostart
```

**If using Container:**
```bash
docker exec -it nova_compute virsh list --all --autostart
```

**Success Criteria:**
Your instance ID or Name should appear in the list. Even if the state is `shut off`, being in this list means it **will** start automatically on the next boot.

### Or Verify Another Way 

Get your instance UUID:

```
openstack server show <INSTANCE_NAME> -c id -f value
```
Check the Libvirt XML definition (Run on Host):

```
virsh dumpxml <INSTANCE_UUID> | grep -A 5 "<on_poweroff>"
```

**Expected Behavior:**

- `<on_poweroff>preserve</on_poweroff>` (Good)
- `<on_crash>preserve</on_crash>` (Good)
- If it says `destroy`, the VM will not resume.

_Note: Nova usually manages this automatically when `resume_instances=true`. If it is wrong, Nova will correct it upon the next state change once the config is fixed._

Ensure Libvirt Service is Enabled

Ensure the host's Libvirt service starts on boot:

```
systemctl enable libvirtd
systemctl status libvirtd
```

### Or Verify Another Way 

Confirm that the property has been applied successfully.
```bash
openstack server show <instance-name-or-id> -c properties
```
*Expected Output:*
```text
| properties   | auto_start='True' |
```

---

## 5. Solution 3: Manual Configuration (Fastest Workaround)
If you want to avoid running the full Ansible playbook right now, or if Solution 1 failed to apply the config, you can manually edit the configuration file. This is often faster for single-node labs.

### Step 5.1: Edit the Configuration File on Host
Kolla mounts configuration files from the host into the container. Edit the file directly on the host:

```bash
nano /etc/kolla/nova-compute/nova.conf
```
*Note: If this file doesn't exist, check `/etc/kolla/nova/nova.conf`. Usually, Kolla manages `/etc/kolla/<service>/`.*

### Step 5.2: Add Resume Parameter
Add the following line under the `[DEFAULT]` section:

```ini
[DEFAULT]
# ... other existing settings ...
resume_instances = true
```
*Save and exit (Ctrl+O, Enter, Ctrl+X).*

### Step 5.3: Restart the Nova Compute Container
Find the container name and restart it to apply changes:

```bash
docker ps | grep nova_compute
docker restart nova_compute
```

### Step 5.4: Verify Logs
Check the logs to ensure it started with the new config:

```bash
docker logs --tail 50 nova_compute
```
Look for a line indicating `resume_instances = True` or no errors during startup.

---

## 6. Testing the Setup
Follow these steps to verify everything works:

1. **Ensure Instance is Running:**
   ```bash
   openstack server list
   # Status should be ACTIVE
   ```

2. **Reboot the Physical/Virtual Server:**
   ```bash
   reboot
   ```

3. **Wait for Services to Start:**
   After the server comes back up, wait 2-3 minutes for Docker and Kolla services to initialize.
   ```bash
   docker ps | grep nova
   ```

4. **Check Instance Status:**
   ```bash
   openstack server list
   ```
   The status should eventually return to `ACTIVE`. You can also try to ping the instance IP.

---

## 7. Troubleshooting

### Issue: Instance stays in "SHUTOFF" state after reboot.
**Fix:**
1. Check if the Libvirt autostart flag is actually set:
   ```bash
   virsh list --all --autostart
   ```
   If your VM is missing, repeat **Solution 2**.

2. Check Nova logs for errors:
   ```bash
   docker logs --tail 100 nova_compute
   ```
   Look for "Failed to resume instance" errors.

3. Manually trigger a start (if auto-start fails once):
   ```bash
   openstack server start <INSTANCE_NAME>
   ```

### Issue: `virsh` command not found.
**Fix:**
Install the libvirt clients package:
```bash
apt install libvirt-clients -y
```

### Issue: `kolla-ansible` command not found.
**Fix:**
Ensure you have activated the Kolla virtual environment:
```bash
source /usr/share/kolla-ansible/venv/bin/activate
# OR
source /opt/venv-kolla/bin/activate
```

---

## Summary of Files & Commands
| Context | File / Command | Purpose |
| :--- | :--- | :--- |
| **Ansible Config** | `/etc/kolla/globals.yml` | Enable `nova_compute_resume_instances` |
| **Manual Config** | `/etc/kolla/nova-compute/nova.conf` | Manually set `resume_instances = true` |
| **Libvirt Tool** | `apt install libvirt-clients -y` | Install `virsh` command |
| **Autostart Flag** | `virsh autostart <UUID>` | Mark VM to start on host boot |
| **Service** | `docker restart nova_compute` | Apply manual config changes |

By combining **Solution 1** (Nova Config) and **Solution 2** (Libvirt Flag), your OpenStack single-node environment will behave like a robust cloud platform, automatically recovering your instances after any planned or unplanned reboot.

---

