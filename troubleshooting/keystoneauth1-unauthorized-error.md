# OpenStack Kolla-Ansible 401 Error Fix Guide
## (After Admin Password Change)

### Table of Contents
1. [Problem Analysis](#1-problem-analysis)
2. [Solution 1: Update `admin-openrc.sh` (Most Likely Fix)](#2-solution-1-update-admin-openrcsh--most-likely-fix)
3. [Solution 2: Update Kolla Ansible `passwords.yml`](#3-solution-2-update-kolla-ansible-passwordsyml)
4. [Solution 3: Reset Keystone Token Cache](#4-solution-3-reset-keystone-token-cache)
5. [Verification Steps](#5-verification-steps)
6. [Re-run Ansible](#6-re-run-ansible)

---

## 1. Problem Analysis
**Error Message:**
```
FAILED - RETRYING: [localhost]: nova | Creating/deleting services (1 retries left).
An exception occurred during task execution. To see the full traceback, use -vvv. The error was: keystoneauth1.exceptions.http.Unauthorized: The request you have made requires authentication. (HTTP 401) (Request-ID: req-c976b512-3e40-4acd-9fb4-5f44c77d99dc)
```
**Error Message:** `keystoneauth1.exceptions.http.Unauthorized: The request you have made requires authentication. (HTTP 401)`

**Cause:**
You changed the **Admin User password** via the OpenStack Dashboard (Horizon). However, **Kolla-Ansible** still holds the **old password** in two places:
1.  The environment variable file (`admin-openrc.sh`) used by your current shell session.
2.  The Kolla Ansible secrets file (`passwords.yml`) used by the automation tool to authenticate with Keystone.

Because Kolla-Ansible is trying to log in with the old password, Keystone rejects the request with a **401 Unauthorized** error.

---

## 2. Solution 1: Update `admin-openrc.sh` (Most Likely Fix)
Your current terminal session is likely using the old credentials. You must update the source file and reload it.

### Step 2.1: Edit the File
Open the admin OpenRC file:
```bash
nano /etc/kolla/admin-openrc.sh
```

### Step 2.2: Update Password
Find the line starting with `export OS_PASSWORD=` and change the value to your **new password**.

```bash
export OS_USERNAME=admin
export OS_PASSWORD=YOUR_NEW_PASSWORD  <-- Update this
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://192.168.68.150:5000/v3
export OS_IDENTITY_API_VERSION=3
```
*Save and exit (Ctrl+O, Enter, Ctrl+X).*

### Step 2.3: Reload Environment
Source the file again to apply changes to your current session:
```bash
source /etc/kolla/admin-openrc.sh
```

### Step 2.4: Test Manual Login
Verify the new password works manually before running Ansible:
```bash
openstack token issue
```
*If this returns a token table, your CLI is fixed. If it fails, check the password again.*

---

## 3. Solution 2: Update Kolla Ansible `passwords.yml`
Even if your CLI works, **Kolla-Ansible** reads from its own secure password file (`passwords.yml`). If this file has the old password, the playbook will fail.

### Option A: Edit Manually (Fastest)
1. Open the passwords file:
   ```bash
   nano /etc/kolla/passwords.yml
   ```

2. Find the key named `keystone_admin_password`. It usually looks like this:
   ```yaml
   keystone_admin_password: "OLD_RANDOM_PASSWORD"
   ```

3. Change the value to your **new password**:
   ```yaml
   keystone_admin_password: "YOUR_NEW_PASSWORD"
   ```
   *Note: Ensure the password is wrapped in quotes if it contains special characters.*

4. Save and exit.

### Option B: Regenerate Specific Password (Alternative)
If you prefer not to edit manually, you can use the `kolla-genpwd` tool, but be careful as it might generate a *new random* password instead of setting your specific chosen one. Since you already chose a password in the dashboard, **Option A (Manual Edit)** is safer to keep them in sync.

---

## 4. Solution 3: Reset Keystone Token Cache
Sometimes, Docker containers or services cache old tokens. Restarting the Keystone container ensures it picks up the new credential validation immediately.

### Step 4.1: Restart Keystone Container
```bash
docker restart keystone
```

### Step 4.2: Verify Service Status
Wait 10 seconds, then check if Keystone is healthy:
```bash
docker ps | grep keystone
```

---

## 5. Verification Steps
Before running the heavy Ansible playbook, confirm everything is synchronized.

1. **Check Environment Variable:**
   ```bash
   echo $OS_PASSWORD
   # Output should be your NEW password
   ```

2. **Check Passwords File Consistency:**
   ```bash
   grep keystone_admin_password /etc/kolla/passwords.yml
   # Output should match your NEW password
   ```

3. **Test OpenStack CLI Again:**
   ```bash
   openstack server list
   # Should list your instances without error
   ```

---

## 6. Re-run Ansible
Now that the passwords are synchronized, run the reconfigure command again.

### Command
```bash
kolla-ansible -i all-in-one reconfigure --tags nova
```

*If you updated `passwords.yml`, Kolla-Ansible will now successfully authenticate with Keystone using the new password and update the Nova configuration.*

### Summary of Files Changed
| File Name | Purpose | Action Required |
| :--- | :--- | :--- |
| `admin-openrc.sh` | CLI Environment | Update `OS_PASSWORD` |
| `passwords.yml` | Ansible Secrets | Update `keystone_admin_password` |

### Troubleshooting Tips
- **Still getting 401?** Ensure there are no extra spaces in `passwords.yml` around the password value.
- **Special Characters?** If your new password contains `$`, `"`, or `'`, ensure they are properly escaped or wrapped in single quotes in `passwords.yml`.
- **Dashboard Access?** After fixing, try logging into the Horizon Dashboard again to ensure the password change didn't lock out other services.

By following these steps, your Kolla-Ansible playbook will authenticate successfully, and you can proceed with enabling the `resume_instances` feature.
