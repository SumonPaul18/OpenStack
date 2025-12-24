```
sudo ovs-vsctl list-ports br-ex
```
#### Output like bellow

    enp4s0
    phy-br-ex

```
cat /etc/kolla/globals.yml
```
---

```
# You can use this file to override _any_ variable throughout Kolla.
# Additional options can be found in the
# 'kolla-ansible/ansible/group_vars/all.yml' file. Default value of all the
# commented parameters are shown here, To override the default value uncomment
# the parameter and change its value.

# Dummy variable to allow Ansible to accept this file.
workaround_ansible_issue_8743: yes

###################
# Ansible options
###################

# Base OS for containers
kolla_base_distro: "ubuntu"

# OpenStack release
kolla_install_type: "source"

# Management interface (host-only)
network_interface: "enp1s0"

# External interface for Neutron (bridged, no IP!)
neutron_external_interface: "enp4s0"
neutron_bridge_mappings: "physnet1:enp4s0"
# Virtual IP for internal traffic (must be unused)
kolla_internal_vip_address: "192.168.0.250"

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
#enable_heat: "yes"

```

#### এই neutron_bridge_mappings: "physnet1:enp4s0" অংশটি আমাকে সহজভাবে ধাপে-ধাপে প্রথম থেকে  বোঝান এবং ভেরিফাই করার গাইড দিন।  
