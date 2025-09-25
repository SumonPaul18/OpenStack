# 🌩️ OpenStack Deployment & Management Guide  
### Build Your Own Private or Public Cloud with Confidence  

> **🎯 Audience**: Cloud Engineers • DevOps Professionals • System Administrators • IT Students • OpenStack Enthusiasts  
> **📅 Last Updated**: April 2025  
> **🧑‍💻 Author**: Sumon Paul • [**Cloud Engineer** YouTube Channel](https://www.youtube.com/@cloudengineer187)  
> **🔖 Tags**: `#OpenStack` `#PrivateCloud` `#KollaAnsible` `#PackStack` `#CloudComputing` `#IaaS`

![OpenStack Banner](https://docs.openstack.org/www/assets/images/openstack-logo-full.svg)

---

## 📚 Table of Contents

- [🌟 About This Repository](#-about-this-repository)
- [☁️ What is OpenStack?](#️-what-is-openstack)
  - [Why OpenStack?](#why-openstack)
  - [History & Evolution](#history--evolution)
  - [Core Components](#core-components)
  - [Pros & Cons](#pros--cons)
  - [OpenStack vs Alternatives](#openstack-vs-alternatives)
- [⚙️ Prerequisites for Installation](#️-prerequisites-for-installation)
- [🚀 OpenStack Deployment Methods Overview](#-openstack-deployment-methods-overview)
  - [Official & Community-Supported Tools](#official--community-supported-tools)
  - [Methods Covered in This Repository](#methods-covered-in-this-repository)
- [🔧 Post-Installation: Verification & Essential Operations](#-post-installation-verification--essential-operations)
  - [How to Verify Your Installation](#how-to-verify-your-installation)
  - [Accessing OpenStack](#accessing-openstack)
  - [Managing via CLI – Essential Commands](#managing-via-cli--essential-commands)
  - [Critical Tips for Configuration, Operation & Maintenance](#critical-tips-for-configuration-operation--maintenance)
- [📄 License](#-license)
- [📬 Feedback & Contributions](#-feedback--contributions)

---

## 🌟 About This Repository

This repository provides **hands-on, production-ready guides** for deploying **OpenStack** using **three widely used methods**:

1. **Kolla-Ansible** – Containerized (Docker + Ansible), ideal for production.
2. **PackStack** – Rapid All-in-One setup via Puppet (RDO), perfect for labs.
3. **Manual Installation** – Step-by-step Ubuntu-based guide for deep learning.

Whether you're building a **private cloud**, running a **home lab**, or preparing for **certification**, this repo gives you everything from theory to terminal commands — with real-world fixes and troubleshooting.

---

## ☁️ What is OpenStack?

### Why OpenStack?
OpenStack is an **open-source Infrastructure-as-a-Service (IaaS)** platform that lets you build and manage **private and public clouds** using standard hardware. Used by **NASA, CERN, AT&T**, and thousands of enterprises.

### History & Evolution
- **Launched in 2010** by **NASA** and **Rackspace**.
- Now governed by the **Open Infrastructure Foundation (OpenInfra)**.
- Enables **vendor-neutral**, **scalable**, and **secure** cloud infrastructure.

### Core Components
| Category | Service | Purpose |
|--------|--------|--------|
| Compute | Nova | VM lifecycle |
| Networking | Neutron | SDN |
| Storage | Cinder (block), Swift (object) | Persistent storage |
| Identity | Keystone | Auth & authz |
| Image | Glance | VM image registry |
| Dashboard | Horizon | Web UI |
| Orchestration | Heat | IaC |

### Pros & Cons

| ✅ Pros | ❌ Cons |
|--------|--------|
| Open-source & free | Steep learning curve |
| Highly modular & scalable | Complex networking |
| No vendor lock-in | Requires skilled team |
| Supports hybrid/multi-cloud | Resource-intensive |

### OpenStack vs Alternatives

| Platform | Best For |
|--------|--------|
| **OpenStack** | On-prem private/public cloud with full control |
| **AWS / Azure / GCP** | Public cloud with zero infra management |
| **Proxmox VE** | Small-scale virtualization |
| **VMware vCloud** | Existing VMware environments |

> 💡 **Use OpenStack when**: You need **data sovereignty**, **custom billing**, or to **offer white-label cloud services**.

---

## ⚙️ Prerequisites for Installation

### Hardware (Minimum for All-in-One)
- **CPU**: 2+ cores (4+ recommended)
- **RAM**: 8 GB (16+ GB for production)
- **Disk**: 40+ GB SSD (100+ GB preferred)
- **Network**: 2–3 NICs (Management, Data, External)

### Software Requirements
- **OS**: Ubuntu 22.04/24.04, Rocky Linux 8/9, AlmaLinux 8/9
- **Virtualization**: Enabled in BIOS (`vmx`/`svm`)
- **Root or sudo access**
- **Static IP configuration**
- **SELinux disabled** (for RHEL-based)
- **Firewall disabled** (or required ports opened)

> 🔧 **Tip**: Use **VirtualBox** or **Proxmox** for labs. For production, use bare-metal servers.

---

## 🚀 OpenStack Deployment Methods Overview

OpenStack can be deployed in **many ways**, depending on your use case, scale, and expertise. Below is a comprehensive list of **official and community-supported deployment tools**.

### Official & Community-Supported Tools

| Tool | Type | Best For | Official Docs |
|------|------|--------|--------------|
| **Kolla-Ansible** | Containerized (Docker + Ansible) | Production, HA, scalable | [docs.openstack.org/kolla-ansible](https://docs.openstack.org/kolla-ansible/) |
| **PackStack** | Puppet-based (RDO) | Quick AIO labs, CentOS/AlmaLinux | [rdoproject.org/install/packstack](https://www.rdoproject.org/install/packstack/) |
| **DevStack** | Shell script (development) | Developers, testing | [docs.openstack.org/devstack](https://docs.openstack.org/devstack/latest/) |
| **OpenStack Charms (Juju)** | Model-driven (Ubuntu) | Canonical/Ubuntu environments | [ubuntu.com/openstack/install](https://ubuntu.com/openstack/install) |
| **TripleO (OpenStack on OpenStack)** | Production-grade (Red Hat) | Large-scale, enterprise | [docs.openstack.org/tripleo](https://docs.openstack.org/tripleo/latest/) |
| **Helm Charts (Airship)** | Kubernetes-native | Cloud-native deployments | [airshipit.org](https://airshipit.org/) |
| **Manual Installation** | Step-by-step | Learning, customization | [docs.openstack.org/install-guide](https://docs.openstack.org/install-guide/) |

> 📌 **Note**: While **DevStack** and **TripleO** are powerful, they are **not included** in this repo due to scope.

### Methods Covered in This Repository

✅ **1. Kolla-Ansible (Containerized, Production-Ready)**  
📁 Path: [`/deployment-tools/kolla-ansible/`](deployment-tools/kolla-ansible/)  
📘 Guides:
- [Install OpenStack AIO with Kolla-Ansible](deployment-tools/kolla-ansible/install-os-aio-kolla-ansible.md)
- [Complete Kolla-Ansible v2 Guide](deployment-tools/kolla-ansible/openstack-aio-v2.md)

✅ **2. PackStack (RDO-based, All-in-One)**  
📁 Path: [`/deployment-tools/packstack/`](deployment-tools/packstack/)  
📘 Guides:
- [Install-OpenStack-PackStack-Using-Shell-Script](https://github.com/SumonPaul18/openstack-packstack)
- [Install OpenStack on AlmaLinux using PackStack (.txt)](deployment-tools/packstack/Install%20OpenStack%20on%20AlmaLinux%20using%20PackStack.txt)
- [Install OpenStack-PackStack on AlmaLinux 9](deployment-tools/packstack/Install%20OpenStack-PackStack%20on%20AlmaLinux%209.txt)
- [PDF Version (Offline)](deployment-tools/packstack/Install%20OpenStack%20on%20AlmaLinux%20using%20PackStack.pdf)

✅ **3. Manual Installation (Step-by-Step, Educational)**  
📁 Path: [`/manually/`](manually/)  
📘 Guides:
- [Manual Installation Guide (Part 1)](manually/install-openstack-manually-1.md)
- [Automated Shell Scripts (Part 2)](manually/install-openstack-manually-2.md)

---

## 🔧 Post-Installation: Verification & Essential Operations

### How to Verify Your Installation

After installation, **always verify** that all services are running:

```bash
# Source admin credentials
source ~/keystonerc_admin        # PackStack
# OR
source /etc/kolla/admin-openrc.sh  # Kolla-Ansible

# Check core services
openstack service list

# Check compute & network agents
openstack compute service list
openstack network agent list

# Verify hypervisors
openstack hypervisor list

# Test token issuance
openstack token issue
```

✅ **Success**: All services show `enabled` and `up`.

---

### Accessing OpenStack

- **Horizon Dashboard**: `http://<your-ip>/dashboard`
- **Default Login**:
  - **Username**: `admin`
  - **Password**: From `~/keystonerc_admin` or `/etc/kolla/passwords.yml`
- **Get Password**:
  ```bash
  grep keystone_admin_password /etc/kolla/passwords.yml
  ```

> 🔒 **Security Tip**: Change default passwords and disable demo projects in production.

---

### Managing via CLI – Essential Commands

Install OpenStack CLI (if not installed):
```bash
pip install python-openstackclient
```

#### 🔹 Identity & Projects
```bash
openstack project list
openstack user list
openstack role assignment list --names
```

#### 🔹 Compute
```bash
openstack server list
openstack server create --image <img> --flavor <flavor> --network <net> test-vm
openstack server delete <server-id>
openstack flavor list
openstack hypervisor stats show
```

#### 🔹 Networking
```bash
openstack network list
openstack subnet list
openstack router list
openstack floating ip list
```

#### 🔹 Images & Volumes
```bash
openstack image list
openstack volume list
```

#### 🔹 System Health
```bash
openstack catalog list
openstack endpoint list
nova-status upgrade check
```

> 💡 **Pro Tip**: Use `--format json` or `--format yaml` for scripting.

---

### Critical Tips for Configuration, Operation & Maintenance

#### 🔸 1. **Always Use Version-Pinned Installations**
- Use **stable branches** (e.g., `stable/2025.1`) for Kolla-Ansible.
- Avoid `master` in production.

#### 🔸 2. **Backup Critical Files**
- **Kolla-Ansible**: Backup `/etc/kolla/`, `passwords.yml`, and inventory.
- **PackStack**: Backup `~/keystonerc_admin` and `/root/answers.txt`.

#### 🔸 3. **Enable Instance Auto-Start on Reboot**
Edit `/etc/nova/nova.conf`:
```ini
[DEFAULT]
resume_guests_state_on_host_boot = true
```
Then restart:
```bash
systemctl restart openstack-nova-compute
```

#### 🔸 4. **Fix Common Runtime Issues**
- **Kolla-Ansible**: Install missing modules in Ansible runtime:
  ```bash
  /opt/ansible-runtime/bin/pip install docker dbus-python
  ```
- **PackStack**: If Horizon fails, restart:
  ```bash
  systemctl restart httpd memcached
  ```

#### 🔸 5. **Monitor Logs Proactively**
```bash
# Compute issues
tail -f /var/log/nova/nova-compute.log

# Networking issues
tail -f /var/log/neutron/server.log

# Authentication issues
tail -f /var/log/keystone/keystone.log
```

#### 🔸 6. **Use Screen or tmux for Long-Running Tasks**
```bash
dnf install screen -y
screen -S openstack-install
# (Detach with Ctrl+A, D; Reattach with `screen -r`)
```

#### 🔸 7. **Clean Up Unused Resources**
```bash
# Delete orphaned volumes
openstack volume list --status error -c ID -f value | xargs openstack volume delete

# Remove unused images
openstack image list --status active
```

---

## 📄 License

- This repository is licensed under the **MIT License**.
- OpenStack itself is licensed under **Apache 2.0**.
- See [`LICENSE`](deployment-tools/packstack/LICENSE) for details.

---

## 📬 Feedback & Contributions

Found a typo? Have an improvement?  
👉 **Open an Issue** or **Submit a Pull Request**!

🔗 **Connect**:
- YouTube: [Cloud Engineer by Sumon Paul](https://youtube.com/@CloudEngineer)
- GitHub: [@SumonPaul18](https://github.com/SumonPaul18)

---

> ✨ **Happy Cloud Building!**  
> *"The future is open, scalable, and in your control."*
