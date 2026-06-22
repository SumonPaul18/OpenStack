# OpenStack Instance Auto-Start Failure After Host Reboot
### A Kolla-Ansible Single-Node Troubleshooting Case Study

---

## Table of Contents
- [1. Problem Overview](#1-problem-overview)
- [2. Root Cause](#2-root-cause)
- [3. The Common Mistake](#3-the-common-mistake)
- [4. Correct Solution](#4-correct-solution)
- [5. Step-by-Step Fix](#5-step-by-step-fix)
- [6. Verification](#6-verification)
- [7. Single-Node Race Condition Warning](#7-single-node-race-condition-warning)
- [8. Quick Command Reference](#8-quick-command-reference)

---

## 1. Problem Overview

**Environment:** Single-node OpenStack deployment using Kolla-Ansible (host: `controller`, IP `192.168.68.69`).

**Symptom:** Whenever the host server loses power, crashes, or is rebooted for any reason, all previously running instances (VMs) stay in a **stopped/shutoff** state after the host comes back online. They do not resume automatically, even though they were active before the reboot.

This is **expected default behavior** in OpenStack Nova — not a bug. The fix requires a configuration change plus understanding how Kolla-Ansible handles config files.

---

## 2. Root Cause

Nova Compute has a configuration option called:

```
resume_guests_state_on_host_boot
```

By default this is set to `False`. It controls whether `nova-compute` checks the database for instances that were `ACTIVE` before shutdown and attempts to restart their underlying libvirt domains when the service initializes.

Because this option is `False` by default, `nova-compute` simply ignores the previous VM state on startup — so VMs remain off.

---

## 3. The Common Mistake

A natural first attempt is to edit the file directly inside `/etc/kolla/nova-compute/nova.conf`:

```bash
sudo nano /etc/kolla/nova-compute/nova.conf
```

**This does not work**, because that file is **auto-generated**. Every time `kolla-ansible reconfigure` (or `deploy`) runs, Kolla rebuilds this file from its internal Jinja2 templates plus any custom overrides — completely overwriting manual edits.

| Path | Purpose | Safe to edit? |
|---|---|---|
| `/etc/kolla/nova-compute/nova.conf` | Auto-generated final config | ❌ No — gets overwritten every run |
| `/etc/kolla/config/nova/nova-compute.conf` | Custom override directory | ✅ Yes — this is the intended place |

---

## 4. Correct Solution

Kolla-Ansible supports a **config override mechanism**. Any `.conf` file placed in `/etc/kolla/config/<service>/` is automatically merged into the generated config on every reconfigure — and it persists permanently across future deployments.

For Nova Compute specifically, the override file path is:

```
/etc/kolla/config/nova/nova-compute.conf
```

---

## 5. Step-by-Step Fix

**Step 1 — Create the override directory (if it doesn't exist):**

```bash
sudo mkdir -p /etc/kolla/config/nova
```

**Step 2 — Create/edit the override file:**

```bash
sudo nano /etc/kolla/config/nova/nova-compute.conf
```

**Step 3 — Add the setting:**

```ini
[DEFAULT]
resume_guests_state_on_host_boot = True
```

Save and exit.

**Step 4 — Apply the change with Kolla-Ansible:**

```bash
kolla-ansible -i all-in-one reconfigure --tags nova
```

> ⚠️ Note: `-i all-in-one` (the inventory flag) must come right after `kolla-ansible`, not after the action word `reconfigure`. Correct order matters for reliable command parsing.

---

## 6. Verification

**Check the generated host-side config:**

```bash
grep resume_guests /etc/kolla/nova-compute/nova.conf
```

**Check inside the running container:**

```bash
docker exec -it nova_compute grep "resume_guests_state_on_host_boot" /etc/nova/nova.conf
```

Expected output:

```
resume_guests_state_on_host_boot = True
```

If both commands return this line, the configuration has been applied correctly and will survive future reconfigure runs.

---

## 7. Single-Node Race Condition Warning

On a single-node setup, **MariaDB, RabbitMQ, and Nova all restart together** at boot time. There is a known issue where `nova-compute` can start before MariaDB is fully ready, which silently prevents the auto-resume logic from working — even with the correct setting in place.

**If instances still don't resume after a reboot test:**

1. Wait 2–3 minutes after boot before checking instance status (services need time to initialize).
2. Check that core containers are healthy:
   ```bash
   docker ps -a | grep -E "mariadb|rabbitmq|nova"
   ```
3. Confirm Docker itself is enabled to start on boot:
   ```bash
   systemctl is-enabled docker
   ```
4. Confirm container restart policy is set correctly:
   ```bash
   docker inspect nova_compute --format='{{.HostConfig.RestartPolicy}}'
   ```

**Important:** This setting only resumes instances that were `ACTIVE` (running) at the time of shutdown. Instances manually stopped beforehand will remain off — this is expected behavior, not a bug.

---

## 8. Quick Command Reference

| Task | Command |
|---|---|
| Create override directory | `sudo mkdir -p /etc/kolla/config/nova` |
| Edit override file | `sudo nano /etc/kolla/config/nova/nova-compute.conf` |
| Apply config change | `kolla-ansible -i all-in-one reconfigure --tags nova` |
| Verify in generated config | `grep resume_guests /etc/kolla/nova-compute/nova.conf` |
| Verify inside container | `docker exec -it nova_compute grep "resume_guests_state_on_host_boot" /etc/nova/nova.conf` |
| Check container status | `docker ps -a \| grep -E "mariadb\|rabbitmq\|nova"` |
| Check Docker boot status | `systemctl is-enabled docker` |
| Check restart policy | `docker inspect nova_compute --format='{{.HostConfig.RestartPolicy}}'` |

---

*Case Study Summary: The core fix is a one-line Nova setting (`resume_guests_state_on_host_boot = True`), but it must be placed in the Kolla override directory — not the auto-generated config file — to survive future deployments.*
