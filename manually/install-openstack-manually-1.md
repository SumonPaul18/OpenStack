-----

আপনার হোম ল্যাবে উবুন্টু ওএস-এ OpenStack ম্যানুয়ালি ইনস্টল করার জন্য বিস্তারিত ধাপগুলো নিচে দেওয়া হলো। যেহেতু আপনি DevStack বা Packstack ব্যবহার করতে চাচ্ছেন না এবং প্রথমে একটি মেশিনে ইনস্টল করে পরে অন্য মেশিনে তা প্রসারিত করতে চান, তাই এটি একটি দীর্ঘ প্রক্রিয়া হবে।

## OpenStack ইনস্টলেশনের পূর্বশর্ত এবং প্রস্তুতি

ম্যানুয়াল ইনস্টলেশনের জন্য প্রতিটি মেশিনে কিছু পূর্বশর্ত পূরণ করতে হবে।

### ১. হার্ডওয়্যার এবং নেটওয়ার্কের প্রয়োজনীয়তা

  * **RAM:** প্রতি মেশিনে কমপক্ষে 8 GB RAM (১৬ GB প্রস্তাবিত)
  * **CPU:** মাল্টিকোর প্রসেসর
  * **Storage:** দ্রুত SSD, কমপক্ষে ১০০ GB ফ্রি স্পেস
  * **Network Interfaces:** প্রতিটি নোডে কমপক্ষে দুটি নেটওয়ার্ক ইন্টারফেস কার্ড (NIC) থাকা আবশ্যক। একটি ম্যানেজমেন্ট নেটওয়ার্কের জন্য এবং অন্যটি ডেটা নেটওয়ার্কের জন্য।
      * **Management Network:** OpenStack সার্ভিসগুলোর নিজেদের মধ্যে যোগাযোগের জন্য।
      * **Data/Tenant Network:** ভার্চুয়াল মেশিন এবং বাইরের বিশ্বের সাথে যোগাযোগের জন্য।

### ২. অপারেটিং সিস্টেম প্রস্তুতি

আপনার তিনটি ডেস্কটপ মেশিনে উবুন্টু সার্ভার (LTS ভার্সন প্রস্তাবিত, যেমন Ubuntu 22.04 LTS) ইনস্টল করা আছে ধরে নিচ্ছি। যদি ডেস্কটপ ভার্সন থাকে, তবে কিছু অপ্রয়োজনীয় সার্ভিস বন্ধ করে পারফরম্যান্স অপ্টিমাইজ করা যেতে পারে।

**প্রতিটি মেশিনে করুন:**

  * **সিস্টেম আপডেট:**

    ```bash
    sudo apt update
    sudo apt upgrade -y
    ```

  * **হোস্টনেম সেটআপ:** প্রতিটি মেশিনের জন্য স্বতন্ত্র হোস্টনেম সেট করুন (যেমন: `controller`, `compute1`, `compute2`)।

    ```bash
    sudo hostnamectl set-hostname <your-hostname>
    # উদাহরণস্বরূপ:
    sudo hostnamectl set-hostname controller
    ```

  * **`/etc/hosts` ফাইল কনফিগারেশন:** প্রতিটি মেশিনের `/etc/hosts` ফাইলে অন্য সব মেশিনের IP অ্যাড্রেস এবং হোস্টনেম যোগ করুন যাতে তারা একে অপরের সাথে নাম ব্যবহার করে যোগাযোগ করতে পারে।

    ```bash
    # উদাহরণ: controller নোডে
    127.0.0.1       localhost
    192.168.0.63       controller
    10.0.0.11       compute1
    10.0.0.12       compute2
    ```

    (আপনার নেটওয়ার্কের সাথে মিলিয়ে IP অ্যাড্রেসগুলো পরিবর্তন করুন)

  * **নেটওয়ার্ক কনফিগারেশন:** নেটওয়ার্ক ইন্টারফেসগুলো স্ট্যাটিক IP অ্যাড্রেস ব্যবহার করে কনফিগার করুন। `/etc/netplan/*.yaml` ফাইল এডিট করে এটি করতে পারেন।

    ```yaml
    network:
      version: 2
      renderer: networkd
      ethernets:
        enp0s3:
          dhcp4: no
          addresses: [192.168.0.63/24]
          gateway4: 192.168.0.1
          nameservers:
            addresses: [8.8.8.8, 8.8.4.4]
        enp0s8: 
          dhcp4: no
          addresses: [192.168.1.10/24]
    ```

    পরিবর্তনের পর নেটপ্ল্যান এপ্লাই করুন:

    ```bash
    sudo netplan apply
    ```

  * **NTP সার্ভিস ইনস্টল ও কনফিগারেশন:** সময় সিঙ্ক্রোনাইজেশনের জন্য প্রতিটি নোডে NTP সেটআপ করা অত্যন্ত গুরুত্বপূর্ণ।

    ```bash
    sudo apt install chrony -y
    sudo nano /etc/chrony/chrony.conf
    ```

    সার্ভার লাইনগুলো যোগ বা আনকমেন্ট করুন:

    ```
    server 0.asia.pool.ntp.org iburst maxsources 1
    server 1.asia.pool.ntp.org iburst maxsources 1
    server 2.asia.pool.ntp.org iburst maxsources 2
    
    # অন্যান্য NTP সার্ভার যোগ করতে পারেন
    ```

    সার্ভিস রিস্টার্ট করুন:

    ```bash
    sudo systemctl restart chrony
    sudo systemctl enable chrony
    ```

## ফেজ ১: কন্ট্রোলার নোড ইনস্টলেশন (প্রথম মেশিন)

আপনার প্রথম ডেস্কটপ মেশিনটিকে **কন্ট্রোলার নোড** হিসেবে ব্যবহার করব। এই নোডে OpenStack-এর মূল সার্ভিসগুলো (Keystone, Glance, Nova, Neutron, Horizon, Cinder, Heat ইত্যাদি) ইনস্টল করা হবে।

### ১. ডাটাবেজ ইনস্টল (MariaDB)

Keystone, Nova, Neutron, Glance-এর মতো পরিষেবাগুলির ডেটা সংরক্ষণের জন্য একটি ডাটাবেজ সার্ভার প্রয়োজন।

```bash
sudo apt install mariadb-server python3-pymysql -y
```

**MariaDB কনফিগারেশন:**
`/etc/mysql/mariadb.conf.d/50-server.cnf` ফাইলটি এডিট করুন। `[mysqld]` সেকশনের অধীনে নিচের লাইনগুলো যোগ করুন:

```
[mysqld]
bind-address = 192.168.0.63 # আপনার controller নোডের ম্যানেজমেন্ট IP
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

সেভ করে MariaDB সার্ভিস রিস্টার্ট করুন:

```bash
sudo systemctl restart mariadb
sudo systemctl enable mariadb
```

**সিকিউর ইনস্টলেশন (ঐচ্ছিক কিন্তু প্রস্তাবিত):**

```bash
sudo mysql_secure_installation
```

প্রম্পট অনুযায়ী রুট পাসওয়ার্ড সেট করুন এবং অন্যান্য অপশনগুলো `Y` প্রেস করে সম্পন্ন করুন।

### ২. মেসেজ কিউ ইনস্টল (RabbitMQ)

OpenStack পরিষেবাগুলো একে অপরের সাথে যোগাযোগের জন্য একটি মেসেজ কিউ ব্যবহার করে। RabbitMQ একটি জনপ্রিয় পছন্দ।

```bash
sudo apt install rabbitmq-server -y
```

**RabbitMQ ব্যবহারকারী তৈরি:**

```bash
sudo rabbitmqctl add_user openstack openstack#123 # openstack#123 এর বদলে একটি শক্তিশালী পাসওয়ার্ড দিন
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### ৩. আইডেন্টিটি সার্ভিস ইনস্টল (Keystone)

Keystone OpenStack-এর সমস্ত পরিষেবা এবং ব্যবহারকারীদের জন্য প্রমাণীকরণ (authentication) এবং অনুমোদন (authorization) পরিষেবা সরবরাহ করে।

  * **ডাটাবেজ তৈরি:**

    ```bash
    sudo mysql -u root -p
    CREATE DATABASE keystone;
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'openstack#123';
    GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'openstack#123';
    FLUSH PRIVILEGES;
    EXIT;
    ```

  * **Keystone ইনস্টল:**

    ```bash
    sudo apt install keystone -y
    ```

  * **Keystone কনফিগারেশন:**
    `/etc/keystone/keystone.conf` ফাইলটি এডিট করুন।

    `[database]` সেকশনে:

    ```
    connection = mysql+pymysql://keystone:openstack#123@controller/keystone
    ```

    `[token]` সেকশনে:

    ```
    provider = fernet
    ```

  * **ডাটাবেজ সিঙ্ক:**

    ```bash
    sudo su -s /bin/sh -c "keystone-manage db_sync" keystone
    ```

  * **ফার্নেট কী তৈরি:**

    ```bash
    sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
    sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
    ```

  * **Keystone সার্ভিস রিস্টার্ট:**

    ```bash
    sudo systemctl restart apache2
    sudo systemctl enable apache2
    ```

### ৪. এনভায়রনমেন্ট ভেরিয়েবল সেটআপ

অ্যাডমিন ইউজার হিসেবে OpenStack কমান্ড চালানোর জন্য কিছু এনভায়রনমেন্ট ভেরিয়েবল সেট করতে হবে।

  * **অ্যাডমিন ইউজার তৈরি:**

    ```bash
    export OS_USERNAME=admin
    export OS_PASSWORD=openstack#123
    export OS_PROJECT_NAME=admin
    export OS_USER_DOMAIN_NAME=Default
    export OS_PROJECT_DOMAIN_NAME=Default
    export OS_AUTH_URL=http://controller:5000/v3
    export OS_IDENTITY_API_VERSION=3
    ```

    এই ভেরিয়েবলগুলো একটি `admin-openrc` ফাইলে সেভ করে রাখতে পারেন:

    ```bash
    sudo nano admin-openrc
    ```

    এবং উপরের লাইনগুলো পেস্ট করুন। যখনই অ্যাডমিন কমান্ড চালাতে হবে, তখন `source admin-openrc` কমান্ডটি রান করুন।

  * **প্রজেক্ট, ইউজার এবং রোল তৈরি:**

    ```bash
    openstack domain create --description "An Example Domain" example
    openstack project create --domain default --description "Admin Project" admin
    openstack user create --domain default --password openstack#123 admin
    openstack role create admin
    openstack grant role admin --project admin --user admin

    openstack project create --domain default --description "Service Project" service
    openstack project create --domain default --description "Demo Project" demo
    openstack user create --domain default --password DEMO_PASS demo
    openstack grant role member --project demo --user demo
    ```

### ৫. ইমেজ সার্ভিস ইনস্টল (Glance)

Glance ভার্চুয়াল মেশিন ইমেজ সংরক্ষণের জন্য দায়ী।

  * **ডাটাবেজ তৈরি:**

    ```bash
    sudo mysql -u root -p
    CREATE DATABASE glance;
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'openstack#123';
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'openstack#123';
    FLUSH PRIVILEGES;
    EXIT;
    ```

  * **Glance ইউজার তৈরি:**

    ```bash
    source admin-openrc
    openstack user create --domain default --password openstack#123 glance
    openstack role add --project service --user glance admin
    ```

  * **Glance ইনস্টল:**

    ```bash
    sudo apt install glance -y
    ```

  * **Glance কনফিগারেশন:**
    `/etc/glance/glance-api.conf` এবং `/etc/glance/glance-registry.conf` ফাইলগুলো এডিট করুন।

    `[database]` সেকশনে (উভয় ফাইলে):

    ```
    connection = mysql+pymysql://glance:openstack#123@controller/glance
    ```

    `[glance_store]` সেকশনে (glance-api.conf):

    ```
    stores = file,http
    default_store = file
    filesystem_store_datadir = /var/lib/glance/images/
    ```

    `[keystone_authtoken]` সেকশনে (উভয় ফাইলে, পুরানো অংশ কমেন্ট করে নতুনগুলো যোগ করুন):

    ```
    [keystone_authtoken]
    # ... অন্যান্য লাইন কমেন্ট করুন বা মুছুন
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = glance
    password = openstack#123
    ```

  * **ডাটাবেজ সিঙ্ক:**

    ```bash
    sudo su -s /bin/sh -c "glance-manage db_sync" glance
    ```

  * **সার্ভিস রিস্টার্ট:**

    ```bash
    sudo systemctl restart glance-api glance-registry
    sudo systemctl enable glance-api glance-registry
    ```

  * **ইমেজ আপলোড (পরীক্ষার জন্য):**

    ```bash
    wget http://cloud-images.ubuntu.com/releases/jammy/release/ubuntu-22.04-server-cloudimg-amd64.img
    openstack image create "Ubuntu 22.04" \
      --file ubuntu-22.04-server-cloudimg-amd64.img \
      --disk-format qcow2 --container-format bare \
      --public
    ```

### ৬. কম্পিউট সার্ভিস ইনস্টল (Nova - কন্ট্রোলার পার্ট)

Nova ভার্চুয়াল মেশিনের জীবনচক্র পরিচালনা করে।

  * **ডাটাবেজ তৈরি:**

    ```bash
    sudo mysql -u root -p
    CREATE DATABASE nova_api;
    CREATE DATABASE nova;
    CREATE DATABASE nova_cell0;
    GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'openstack#123';
    GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'openstack#123';
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'openstack#123';
    GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'openstack#123';
    GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'openstack#123';
    GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'openstack#123';
    FLUSH PRIVILEGES;
    EXIT;
    ```

  * **Nova ইউজার তৈরি:**

    ```bash
    source admin-openrc
    openstack user create --domain default --password openstack#123 nova
    openstack role add --project service --user nova admin
    ```

  * **Nova ইনস্টল:**

    ```bash
    sudo apt install nova-api nova-conductor nova-scheduler nova-novncproxy -y
    ```

  * **Nova কনফিগারেশন:**
    `/etc/nova/nova.conf` ফাইলটি এডিট করুন।

    `[api_database]` সেকশনে:

    ```
    connection = mysql+pymysql://nova:openstack#123@controller/nova_api
    ```

    `[database]` সেকশনে:

    ```
    connection = mysql+pymysql://nova:openstack#123@controller/nova
    ```

    `[DEFAULT]` সেকশনে:

    ```
    transport_url = rabbit://openstack:openstack#123@controller
    my_ip = 192.168.0.63 # আপনার controller নোডের ম্যানেজমেন্ট IP
    use_neutron = True
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    ```

    `[api]` সেকশনে:

    ```
    auth_strategy = keystone
    ```

    `[keystone_authtoken]` সেকশনে:

    ```
    # ... অন্যান্য লাইন কমেন্ট করুন বা মুছুন
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = nova
    password = openstack#123
    ```

    `[vnc]` সেকশনে:

    ```
    enabled = True
    server_listen = $my_ip
    server_proxyclient_address = $my_ip
    novncproxy_base_url = http://controller:6080/vnc_auto.html
    ```

    `[glance]` সেকশনে:

    ```
    api_servers = http://controller:9292
    ```

    `[oslo_concurrency]` সে:

    ```
    lock_path = /var/lib/nova/tmp
    ```

  * **ডাটাবেজ সিঙ্ক:**

    ```bash
    sudo su -s /bin/sh -c "nova-manage api_db sync" nova
    sudo su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
    sudo su -s /bin/sh -c "nova-manage db sync" nova
    sudo su -s /bin/sh -c "nova-manage cell_v2 create_cell --name cell1 --verbose" nova
    ```

  * **সার্ভিস রিস্টার্ট:**

    ```bash
    sudo systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy
    sudo systemctl enable nova-api nova-scheduler nova-conductor nova-novncproxy
    ```

### ৭. নেটওয়ার্কিং সার্ভিস ইনস্টল (Neutron - কন্ট্রোলার পার্ট)

Neutron OpenStack-এর নেটওয়ার্কিং সার্ভিস প্রদান করে।

  * **ডাটাবেজ তৈরি:**

    ```bash
    sudo mysql -u root -p
    CREATE DATABASE neutron;
    GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
    GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
    FLUSH PRIVILEGES;
    EXIT;
    ```

  * **Neutron ইউজার তৈরি:**

    ```bash
    source admin-openrc
    openstack user create --domain default --password NEUTRON_PASS neutron
    openstack role add --project service --user neutron admin
    ```

  * **Neutron ইনস্টল:**

    ```bash
    sudo apt install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent -y
    ```

  * **Neutron কনফিগারেশন:**
    `/etc/neutron/neutron.conf` ফাইলটি এডিট করুন।

    `[database]` সেকশনে:

    ```
    connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
    ```

    `[DEFAULT]` সেকশনে:

    ```
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    transport_url = rabbit://openstack:openstack#123@controller
    auth_strategy = keystone
    notify_nova_on_port_status_changes = True
    notify_nova_on_security_group_changes = True
    ```

    `[keystone_authtoken]` সেকশনে:

    ```
    # ... অন্যান্য লাইন কমেন্ট করুন বা মুছুন
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    ```

    `[nova]` সেকশনে:

    ```
    auth_url = http://controller:5000
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    region_name = RegionOne
    project_name = service
    username = nova
    password = openstack#123
    ```

    `/etc/neutron/plugins/ml2/ml2_conf.ini` ফাইলটি এডিট করুন।

    `[ml2]` সেকশনে:

    ```
    type_drivers = flat,vlan,vxlan
    tenant_network_types = vxlan
    mechanism_drivers = linuxbridge,l2population
    extension_drivers = port_security
    ```

    `[ml2_type_flat]` সেকশনে:

    ```
    flat_networks = provider
    ```

    `[ml2_type_vxlan]` সেকশনে:

    ```
    vni_ranges = 1:1000
    ```

    `[securitygroup]` সেকশনে:

    ```
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    ```

    `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` ফাইলটি এডিট করুন।

    `[linux_bridge]` সেকশনে:

    ```
    physical_interface_mappings = provider:enp0s8 
    ```

    `[vxlan]` সেকশনে:

    ```
    enable_vxlan = True
    local_ip = 192.168.0.63
    l2_population = True
    ```

    `[securitygroup]` সেকশনে:

    ```
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
    ```

    `/etc/neutron/l3_agent.ini` ফাইলটি এডিট করুন।

    `[DEFAULT]` সেকশনে:

    ```
    interface_driver = linuxbridge
    ```

    `/etc/neutron/dhcp_agent.ini` ফাইলটি এডিট করুন।

    `[DEFAULT]` সেকশনে:

    ```
    interface_driver = linuxbridge
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    enable_isolated_metadata = True
    ```

    `/etc/neutron/metadata_agent.ini` ফাইলটি এডিট করুন।

    `[DEFAULT]` সেকশনে:

    ```
    nova_metadata_host = controller
    metadata_proxy_shared_secret = METADATA_SECRET # একটি শক্তিশালী পাসওয়ার্ড দিন
    ```

    **Nova-Neutron ইন্টিগ্রেশন:**
    `/etc/nova/nova.conf` ফাইলে `[neutron]` সেকশনে নিচের লাইনগুলো যোগ করুন:

    ```
    [neutron]
    url = http://controller:9696
    auth_url = http://controller:5000
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    region_name = RegionOne
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    service_metadata_proxy = True
    metadata_proxy_shared_secret = METADATA_SECRET
    ```

  * **ডাটাবেজ সিঙ্ক:**

    ```bash
    sudo su -s /bin/sh -c "neutron-manage db sync" neutron
    ```

  * **সার্ভিস রিস্টার্ট:**

    ```bash
    sudo systemctl restart neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent
    sudo systemctl enable neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent
    sudo systemctl restart nova-api # Nova-Neutron ইন্টিগ্রেশনের জন্য এটিও রিস্টার্ট করুন
    ```

### ৮. ড্যাশবোর্ড ইনস্টল (Horizon)

Horizon হচ্ছে OpenStack-এর ওয়েব-ভিত্তিক ইউজার ইন্টারফেস।

  * **Horizon ইনস্টল:**

    ```bash
    sudo apt install openstack-dashboard -y
    ```

  * **Horizon কনফিগারেশন:**
    `/etc/openstack-dashboard/local_settings.py` ফাইলটি এডিট করুন।

    `OPENSTACK_HOST` পরিবর্তন করুন:

    ```python
    OPENSTACK_HOST = "controller"
    ```

    `ALLOWED_HOSTS` এ আপনার কন্ট্রোলার IP বা ডোমেইন যোগ করুন:

    ```python
    ALLOWED_HOSTS = ['controller', '192.168.0.63'] # আপনার কন্ট্রোলার IP
    ```

    `TIME_ZONE` সেট করুন:

    ```python
    TIME_ZONE = "Asia/Dhaka" # আপনার টাইমজোন
    ```

    `session_engine` এ কিছু সিক্রেট কি সেট করুন।

    ```python
    SECRET_KEY = 'a_very_long_and_random_string_of_characters_for_security'
    ```

    `OPENSTACK_KEystone_URL` নিশ্চিত করুন:

    ```python
    OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
    ```

    `OPENSTACK_KEYSTONE_DEFAULT_DOMAIN` সেট করুন:

    ```python
    OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
    ```

    `OPENSTACK_API_VERSIONS` সেট করুন:

    ```python
    OPENSTACK_API_VERSIONS = {
        "identity": 3,
        "image": 2,
        "compute": 2,
        "volume": 2,
        "network": 2,
    }
    ```

  * **Apache সার্ভিস রিস্টার্ট:**

    ```bash
    sudo systemctl reload apache2
    ```

-----

## ফেজ ২: কম্পিউট নোড ইনস্টলেশন (দ্বিতীয় এবং তৃতীয় মেশিন)

আপনার বাকি দুটি ডেস্কটপ মেশিনকে **কম্পিউট নোড** হিসেবে ব্যবহার করব। প্রতিটি কম্পিউট নোডে Nova Compute, Neutron Linuxbridge Agent ইনস্টল করতে হবে।

**প্রতিটি কম্পিউট নোডে নিম্নলিখিত ধাপগুলো অনুসরণ করুন (যেমন compute1, compute2):**

### ১. Nova Compute ইনস্টল

  * **Nova ইনস্টল:**

    ```bash
    sudo apt install nova-compute -y
    ```

  * **Nova কনফিগারেশন:**
    `/etc/nova/nova.conf` ফাইলটি এডিট করুন।

    `[DEFAULT]` সেকশনে:

    ```
    transport_url = rabbit://openstack:openstack#123@controller
    my_ip = 10.0.0.11 # এই কম্পিউট নোডের ম্যানেজমেন্ট IP (যেমন compute1 এর জন্য)
    use_neutron = True
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
    ```

    `[api]` সেকশনে:

    ```
    auth_strategy = keystone
    ```

    `[keystone_authtoken]` সেকশনে:

    ```
    # ... অন্যান্য লাইন কমেন্ট করুন বা মুছুন
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = nova
    password = openstack#123
    ```

    `[vnc]` সেকশনে:

    ```
    enabled = True
    server_listen = 0.0.0.0
    server_proxyclient_address = 10.0.0.11 # এই কম্পিউট নোডের ম্যানেজমেন্ট IP
    novncproxy_base_url = http://controller:6080/vnc_auto.html
    ```

    `[glance]` সেকশনে:

    ```
    api_servers = http://controller:9292
    ```

    `[oslo_concurrency]` সেকশনে:

    ```
    lock_path = /var/lib/nova/tmp
    ```

    `[libvirt]` সেকশনে:

    ```
    virt_type = qemu # যদি আপনার CPU ভার্চুয়ালাইজেশন সমর্থন না করে, অন্যথায় kvm
    ```

  * **সার্ভিস রিস্টার্ট:**

    ```bash
    sudo systemctl restart nova-compute
    sudo systemctl enable nova-compute
    ```

### ২. Neutron Linuxbridge Agent ইনস্টল

  * **Neutron ইনস্টল:**

    ```bash
    sudo apt install neutron-linuxbridge-agent -y
    ```

  * **Neutron কনফিগারেশন:**
    `/etc/neutron/neutron.conf` ফাইলটি এডিট করুন।

    `[DEFAULT]` সেকশনে:

    ```
    transport_url = rabbit://openstack:openstack#123@controller
    auth_strategy = keystone
    ```

    `[keystone_authtoken]` সেকশনে:

    ```
    # ... অন্যান্য লাইন কমেন্ট করুন বা মুছুন
    www_authenticate_uri = http://controller:5000
    auth_url = http://controller:5000
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = Default
    user_domain_name = Default
    project_name = service
    username = neutron
    password = NEUTRON_PASS
    ```

    `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` ফাইলটি এডিট করুন।

    `[linux_bridge]` সেকশনে:

    ```
    physical_interface_mappings = provider:enp0s8 # আপনার ডেটা নেটওয়ার্ক ইন্টারফেসের নাম
    ```

    `[vxlan]` সেকশনে:

    ```
    enable_vxlan = True
    local_ip = 10.0.0.11 # এই কম্পিউট নোডের ম্যানেজমেন্ট IP
    l2_population = True
    ```

    `[securitygroup]` সেকশনে:

    ```
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
    ```

  * **সার্ভিস রিস্টার্ট:**

    ```bash
    sudo systemctl restart neutron-linuxbridge-agent
    sudo systemctl enable neutron-linuxbridge-agent
    ```

-----

## ইনস্টলেশন যাচাই এবং পোস্ট-ইনস্টলেশন কনফিগারেশন

কন্ট্রোলার নোড থেকে নিম্নলিখিত কমান্ডগুলো চালিয়ে ইনস্টলেশন যাচাই করুন:

```bash
source admin-openrc
openstack service list
openstack compute service list
openstack network agent list
```

আপনার সমস্ত OpenStack পরিষেবা `Up` অবস্থায় আছে কিনা এবং `compute` এবং `network` এজেন্টগুলো সঠিক ভাবে দেখাচ্ছে কিনা তা পরীক্ষা করুন।

### নেটওয়ার্ক তৈরি

Horizon ড্যাশবোর্ড (http://controller/) ব্যবহার করে অথবা CLI এর মাধ্যমে নেটওয়ার্ক, সাবনেট এবং রাউটার তৈরি করুন:

  * **প্রোভাইডার নেটওয়ার্ক তৈরি (উদাহরণ):**

    ```bash
    openstack network create --share --external \
      --provider-physical-network provider \
      --provider-network-type flat provider
    ```

  * **প্রোভাইডার সাবনেট তৈরি:**

    ```bash
    openstack subnet create --network provider \
      --allocation-pool start=192.168.1.100,end=192.168.1.150 \
      --dns-nameserver 8.8.8.8 --gateway 192.168.1.1 \
      --no-dhcp --subnet-range 192.168.1.0/24 provider_subnet
    ```

    (আপনার নেটওয়ার্ক অনুযায়ী IP অ্যাড্রেস এবং গেটওয়ে পরিবর্তন করুন)

  * **টেন্যান্ট (প্রাইভেট) নেটওয়ার্ক এবং রাউটার তৈরি:**

    ```bash
    openstack network create private
    openstack subnet create --network private \
      --dns-nameserver 8.8.8.8 --gateway 10.0.0.1 \
      --subnet-range 10.0.0.0/24 private_subnet
    openstack router create router1
    openstack router add subnet router1 private_subnet
    openstack router set router1 --external-gateway provider
    ```

-----

## পরবর্তী দুটি মেশিনে ইনস্টল করার সহজ উপায়

যেহেতু আপনি ম্যানুয়ালি ইনস্টল করেছেন, অন্য দুটি মেশিনে এই প্রক্রিয়াটি সহজ করার জন্য কিছু কৌশল অবলম্বন করতে পারেন:

### ১. স্ক্রিপ্ট ব্যবহার করে অটোমেশন

উপরে বর্ণিত প্রতিটি ধাপের জন্য Bash স্ক্রিপ্ট তৈরি করতে পারেন। উদাহরণস্বরূপ:

  * **`prepare_os.sh`:** হোস্টনেম, `/etc/hosts`, নেটওয়ার্ক এবং NTP কনফিগারেশনের জন্য।
  * **`install_compute_nova.sh`:** Nova Compute ইনস্টল এবং কনফিগারেশনের জন্য।
  * **`install_compute_neutron.sh`:** Neutron Linuxbridge Agent ইনস্টল এবং কনফিগারেশনের জন্য।

এই স্ক্রিপ্টগুলোতে পাসওয়ার্ড, IP অ্যাড্রেস, এবং ইন্টারফেসের নামগুলো ভেরিয়েবল হিসেবে রাখুন, যাতে প্রতিটি নোডের জন্য সহজে পরিবর্তন করা যায়।

### ২. কনফিগারেশন ফাইল কপি

প্রথম কম্পিউট নোড (compute1) সম্পূর্ণ সেটআপ করার পর, `/etc/nova/nova.conf` এবং `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` ফাইলগুলো কপি করে দ্বিতীয় কম্পিউট নোডে (compute2) নিতে পারেন। শুধু IP অ্যাড্রেস এবং হোস্টনেমগুলো নতুন নোডের জন্য ম্যানুয়ালি এডিট করে দিলেই হবে।

**উদাহরণস্বরূপ (compute1 থেকে compute2 তে কপি করার জন্য):**

```bash
# compute1 থেকে
scp /etc/nova/nova.conf user@compute2:/tmp/nova.conf
scp /etc/neutron/plugins/ml2/linuxbridge_agent.ini user@compute2:/tmp/linuxbridge_agent.ini

# compute2 তে
sudo cp /tmp/nova.conf /etc/nova/nova.conf
sudo cp /tmp/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

**গুরুত্বপূর্ণ:** কপি করার পর অবশ্যই ফাইলগুলো এডিট করে **compute2 এর নিজস্ব IP অ্যাড্রেস এবং hostname** সেট করতে ভুলবেন না।

### ৩. ভার্চুয়ালাইজেশন টেমপ্লেট

যদিও আপনার ফিজিক্যাল ডেস্কটপ, যদি ভবিষ্যতে আপনি VM এর উপর OpenStack ইনস্টল করেন, তবে একটি বেস VM তৈরি করে তার স্ন্যাপশট নিয়ে অন্যান্য নোড তৈরি করা সহজ হবে। কিন্তু এটি আপনার বর্তমান পরিস্থিতির জন্য প্রযোজ্য নয়।

### ৪. ক্রমান্বয়ে ইনস্টলেশন

  * **কন্ট্রোলার নোড:** প্রথমে এটি পুরোপুরি সেটআপ করুন।
  * **প্রথম কম্পিউট নোড (compute1):** এটি ইনস্টল করুন এবং নিশ্চিত করুন যে এটি কন্ট্রোলারের সাথে সঠিকভাবে যোগাযোগ করছে।
  * **দ্বিতীয় কম্পিউট নোড (compute2):** একই প্রক্রিয়া অনুসরণ করে এটি ইনস্টল করুন।

প্রতিটি ধাপ সম্পূর্ণ হওয়ার পর, OpenStack ড্যাশবোর্ড এবং CLI ব্যবহার করে আপনার রিসোর্সগুলো ঠিকমতো দেখা যাচ্ছে কিনা তা যাচাই করুন।

ম্যানুয়াল ইনস্টলেশন একটি জটিল প্রক্রিয়া, তাই প্রতিটি কমান্ড সাবধানে অনুসরণ করুন এবং লগ ফাইলগুলো (যেমন `/var/log/nova/nova-api.log`, `/var/log/neutron/neutron-server.log` ইত্যাদি) পরীক্ষা করে সমস্যা সমাধান করুন।

যদি কোনো নির্দিষ্ট ধাপে আটকে যান বা কোনো ত্রুটি পান, তবে আমাকে জিজ্ঞাসা করতে দ্বিধা করবেন না।
