# ✅ **Kolla-Ansible OpenStack Single-Node Installation Guide**  
### **Ubuntu 24.04 | Kolla 2025.1 | 2 NICs | Production-Ready**

> **Hardware**: 4 vCPU, 16 GB RAM, 250 GB SSD  
> **NICs**:  
> - `enp1s0` → Management (IP: `192.168.0.51/24`)  
> - `enp4s0` → External (IP-free, bridged to `br-ex`)  
> **VIP**: `192.168.0.250` (must be unused in your LAN)

---

## 📌 Table of Contents

1. [Preparation](#1-preparation)
2. [System Configuration](#2-system-configuration)
3. [Install Kolla-Ansible](#3-install-kolla-ansible)
4. [Configure Kolla](#4-configure-kolla)
5. [Deploy OpenStack](#5-deploy-openstack)
6. [Post-Deployment Setup](#6-post-deployment-setup)
7. [External Network Setup](#7-external-network-setup)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Preparation

### 1.1 Update System
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 1.2 Install Dependencies
```bash
sudo apt install -y python3-dev libffi-dev gcc libssl-dev python3-venv python3-pip
```

---

## 2. System Configuration

### 2.1 Configure NICs

#### Management NIC (`enp1s0`)
```bash
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.0.51/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    enp4s0:
      dhcp4: no
```

Apply:
```bash
sudo netplan apply
```

### 2.2 Disable Unnecessary Services
```bash
sudo systemctl disable --now ufw
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab
```

### 2.3 Configure Hosts File
```bash
echo "192.168.0.51 $(hostname)" | sudo tee -a /etc/hosts
```

---

## 3. Install Kolla-Ansible

### 3.1 Create Virtual Environment
```bash
python3 -m venv /opt/venv-kolla
source /opt/venv-kolla/bin/activate
pip install --upgrade pip
```

### 3.2 Install Kolla-Ansible
```bash
pip install "kolla-ansible>=2025.1" -c https://releases.openstack.org/constraints/upper/2025.1
```

### 3.3 Copy Configuration Files
```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
cp -r /opt/venv-kolla/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
```

### 3.4 Generate Passwords
```bash
kolla-genpwd
```

---

## 4. Configure Kolla

### 4.1 Configure `/etc/kolla/globals.yml`

```yaml
# Basic Settings
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "2025.1"

# Network Interfaces
network_interface: "enp1s0"                          # Management NIC
neutron_external_interface: "enp4s0"                # External NIC (IP-free!)
kolla_internal_vip_address: "192.168.0.250"         # Must be unused IP
kolla_external_vip_address: "192.168.0.250"         # Same as internal for single-node

# Enable Services
enable_cinder: "no"
enable_haproxy: "yes"
enable_heat: "no"
enable_horizon: "yes"
enable_swift: "no"
enable_manila: "no"
enable_sahara: "no"

# Neutron Settings
enable_neutron: "yes"
neutron_plugin_agent: "openvswitch"
neutron_bridge_mappings: "physnet1:br-ex"           # Critical for external network

# Security
enable_tls: "no"
kolla_enable_tls_external: "no"
kolla_enable_tls_internal: "no"

# Restart Policy (Fixes VIP & Container Issues)
docker_restart_policy: "unless-stopped"
host_network_mode: "yes"

# Nova Auto-Start (Fixes Instance Recovery)
nova_compute_privileged: "yes"
```

> 🔑 **Key Notes**:
> - `neutron_bridge_mappings: "physnet1:br-ex"` → Physical network name is `physnet1`
> - External NIC (`enp4s0`) **MUST have no IP**

### 4.2 Configure Inventory
```bash
# ~/all-in-one
[control]
localhost ansible_connection=local

[network]
localhost ansible_connection=local

[compute]
localhost ansible_connection=local

[monitoring]
localhost ansible_connection=local

[storage]
localhost ansible_connection=local
```

---

## 5. Deploy OpenStack

### 5.1 Bootstrap Servers
```bash
source /opt/venv-kolla/bin/activate
kolla-ansible -i ~/all-in-one bootstrap-servers
```

### 5.2 Run Prechecks
```bash
kolla-ansible -i ~/all-in-one prechecks
```

### 5.3 Deploy
```bash
kolla-ansible -i ~/all-in-one deploy
```

---

## 6. Post-Deployment Setup

### 6.1 Generate OpenRC File
```bash
kolla-ansible post-deploy
```

### 6.2 Source Admin Credentials
```bash
source /etc/kolla/admin-openrc.sh
```

### 6.3 Create Initial Resources
```bash
cd /opt/venv-kolla/share/kolla-ansible/init-runonce
./init-runonce
```

> ⚠️ **Note**: `init-runonce` creates only internal network. External network must be created manually (Section 7).

---

## 7. External Network Setup

### 7.1 Prepare External NIC

```bash
# Flush any IP from external NIC
sudo ip addr flush dev enp4s0
sudo ip link set enp4s0 up
```

### 7.2 Verify OVS Bridge

```bash
# Check if enp4s0 is bridged to br-ex
sudo ovs-vsctl list-ports br-ex
```

> ✅ **Expected Output**:
> ```
> enp4s0
> phy-br-ex
> ```

### 7.3 Create External Network

```bash
# Create provider network (flat type)
openstack network create \
  --external \
  --provider-physical-network physnet1 \
  --provider-network-type flat \
  public

# Create subnet (match your home LAN)
openstack subnet create \
  --network public \
  --subnet-range 192.168.0.0/24 \
  --gateway 192.168.0.1 \
  --no-dhcp \
  public-subnet
```

> 🔑 **Key Notes**:
> - `--provider-physical-network physnet1` → Must match `neutron_bridge_mappings` in `globals.yml`
> - Subnet range = your home LAN range

### 7.4 Configure Router

```bash
# Set external gateway
openstack router set --external-gateway public demo-router

# Enable SNAT for internet access
openstack router set --enable-snat demo-router
```

### 7.5 Restart Neutron Services

```bash
docker restart neutron_server neutron_openvswitch_agent
```

---

## 8. Instance Management

### 8.1 Launch Instance with Floating IP

```bash
# Create keypair
ssh-keygen -t rsa -f ~/.ssh/mykey
openstack keypair create --public-key ~/.ssh/mykey.pub mykey

# Launch instance
openstack server create \
  --image cirros-0.6.2 \
  --flavor demo-flavor \
  --network demo-net \
  --key-name mykey \
  demo-instance

# Assign floating IP
openstack floating ip create public
openstack server add floating ip demo-instance 192.168.0.100
```

### 8.2 Auto-Start Instances on Reboot

#### 8.2.1 Configure Nova
```bash
# /etc/kolla/nova-compute/nova.conf
[DEFAULT]
resume_guests_state_on_host_boot = true
```

#### 8.2.2 Enable Privileged Mode
```bash
# /etc/kolla/globals.d/privileged.yml
nova_compute_privileged: "yes"
```

#### 8.2.3 Apply Changes
```bash
kolla-ansible -i ~/all-in-one deploy
```

---

## 9. Troubleshooting

### 9.1 VIP Not Assigned After Reboot

#### Fix: Ensure restart policy
```bash
# /etc/kolla/globals.d/restart.yml
docker_restart_policy: "unless-stopped"
host_network_mode: "yes"
```

Redeploy:
```bash
kolla-ansible -i ~/all-in-one deploy
```

### 9.2 External Network "Unknown Physical Network"

#### Fix: Verify bridge mappings
```bash
# Check neutron_bridge_mappings
grep neutron_bridge_mappings /etc/kolla/globals.yml

# Check OVS ports
sudo ovs-vsctl list-ports br-ex
```

### 9.3 Instance Can't Access Internet

#### Fix: Enable SNAT
```bash
openstack router set --enable-snat demo-router
```

### 9.4 Horizon Login Issues

#### Fix: Cookie Error
- Use `http://192.168.0.250` instead of HTTPS
- Clear browser cookies
- Check `/etc/kolla/passwords.yml` for admin password

### 9.5 Lost SSH Key

#### Fix: Create new keypair
```bash
ssh-keygen -t rsa -f ~/.ssh/newkey
openstack keypair create --public-key ~/.ssh/newkey.pub newkey
```

---

## 10. Maintenance Commands

### 10.1 Restart Services
```bash
# Restart single service
docker restart nova_compute

# Restart entire stack
kolla-ansible -i ~/all-in-one reconfigure
```

### 10.2 Check Logs
```bash
# Nova logs
docker logs nova_compute

# Neutron logs
docker logs neutron_openvswitch_agent

# Keepalived logs (for VIP)
docker logs keepalived
```

### 10.3 Upgrade OpenStack
```bash
# Update Kolla-Ansible
pip install --upgrade kolla-ansible

# Reconfigure
kolla-ansible -i ~/all-in-one reconfigure
```

---

## 🔚 Final Notes

- **Always backup** `/etc/kolla/` before changes
- **Use CLI for critical operations** (`openstack` commands)
- **Monitor resources**: 16GB RAM is minimum for full stack
- **For production**: Add monitoring (Prometheus/Grafana)

> ✅ **Your OpenStack is now production-ready with external network, auto-recovery, and internet access!**

---

**Author**: Sumon Paul  
**Environment**: Kolla-Ansible 2025.1 | Ubuntu 24.04  
**Last Updated**: December 2025
