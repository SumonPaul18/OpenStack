-----

OpenStack ম্যানুয়াল ইনস্টলেশনের জন্য একটি স্বয়ংক্রিয় শেল স্ক্রিপ্ট তৈরি করা একটি বেশ চ্যালেঞ্জিং কাজ, কারণ প্রতিটি ধাপের জন্য IP অ্যাড্রেস, পাসওয়ার্ড, এবং কিছু কনফিগারেশন প্যারামিটার ব্যবহারকারীর ইনপুট বা পরিবেশের ওপর নির্ভরশীল।

তবে, আমি আপনাকে একটি ফ্রেমওয়ার্ক হিসেবে দুটি প্রধান শেল স্ক্রিপ্ট দেব: একটি **কন্ট্রোলার নোডের জন্য** এবং অন্যটি **কম্পিউট নোডের জন্য**। এই স্ক্রিপ্টগুলো প্রতিটি ধাপের মূল কমান্ডগুলো ধারণ করে। আপনাকে স্ক্রিপ্টগুলো আপনার নির্দিষ্ট নেটওয়ার্ক কনফিগারেশন, পছন্দসই পাসওয়ার্ড, এবং অন্যান্য বিবরণ অনুযায়ী **ম্যানুয়ালি সম্পাদনা করতে হবে**।

**গুরুত্বপূর্ণ নোট:**

  * এই স্ক্রিপ্টগুলো শুধুমাত্র একটি গাইডলাইন। এগুলোতে ত্রুটি থাকতে পারে এবং আপনার নির্দিষ্ট পরিবেশের সাথে মানিয়ে নিতে আরও পরিবর্তন প্রয়োজন হতে পারে।
  * ইনস্টলেশনের সময় ত্রুটি এড়াতে, প্রতিটি প্রধান ধাপের পর `$?` ভেরিয়েবল ব্যবহার করে কমান্ড সফল হয়েছে কিনা তা পরীক্ষা করার সুপারিশ করা হয়। আমি স্ক্রিপ্টে এর কিছু উদাহরণ দেব।
  * পাসওয়ার্ড এবং অন্যান্য সংবেদনশীল তথ্য স্ক্রিপ্টে সরাসরি হার্ডকোড করা হয়েছে। প্রোডাকশন পরিবেশে এটি কাম্য নয়, তবে হোম ল্যাবের জন্য এটি সুবিধাজনক হতে পারে। আপনি চাইলে সেগুলোর জন্য প্রম্পট যোগ করতে পারেন।
  * স্ক্রিপ্ট চালানোর আগে এটি **সম্পূর্ণভাবে পর্যালোচনা করুন এবং প্রতিটি লাইন বুঝুন**।

-----

## ১. স্ক্রিপ্ট চালানোর পূর্বশর্ত

১.  **নেটওয়ার্ক প্রস্তুতি:** নিশ্চিত করুন যে আপনার তিনটি মেশিনের নেটওয়ার্ক ইন্টারফেসগুলো উপরে বর্ণিত ম্যানুয়াল গাইডের মতো সঠিকভাবে কনফিগার করা আছে। প্রতিটি মেশিনের জন্য সঠিক IP অ্যাড্রেস এবং নেটওয়ার্ক ইন্টারফেসের নাম (যেমন `enp0s3`, `enp0s8`) জানতে হবে।
২.  **Ubuntu Server LTS:** প্রতিটি মেশিনে Ubuntu Server LTS ইনস্টল করা আছে ধরে নিচ্ছি।
৩.  **sudo অধিকার:** স্ক্রিপ্টটি `root` ইউজার হিসেবে অথবা `sudo` ব্যবহার করে চালানো হবে।

-----

## ২. কন্ট্রোলার নোড ইনস্টলেশন স্ক্রিপ্ট (`install_controller.sh`)

এই স্ক্রিপ্টটি আপনার কন্ট্রোলার নোডের জন্য। এটি চালানোর আগে আপনাকে আপনার নেটওয়ার্কের বিবরণ, পাসওয়ার্ড এবং ইন্টারফেসের নামগুলো স্ক্রিপ্টে আপডেট করতে হবে।

```bash
#!/bin/bash

# ---------------------------------------------------
# OpenStack Controller Node Installation Script
# Ubuntu LTS
#
# Before running:
# 1. Update variables in the 'Configuration Variables' section.
# 2. Ensure network interfaces are correctly named and configured.
# 3. Run with sudo/root privileges.
# ---------------------------------------------------

set -e # Exit immediately if a command exits with a non-zero status.

# --- Configuration Variables ---
# !!! IMPORTANT: Update these variables according to your environment !!!
CONTROLLER_MGMT_IP="10.0.0.10" # Management IP of this controller node
CONTROLLER_HOSTNAME="controller"
COMPUTE1_HOSTNAME="compute1"
COMPUTE1_MGMT_IP="10.0.0.11"
COMPUTE2_HOSTNAME="compute2"
COMPUTE2_MGMT_IP="10.0.0.12"

MGMT_INTERFACE="enp0s3" # Management network interface name (e.g., enp0s3)
DATA_INTERFACE="enp0s8" # Data/Provider network interface name (e.g., enp0s8)
DEFAULT_GATEWAY="10.0.0.1" # Your network's default gateway
DNS_SERVER="8.8.8.8"     # Your preferred DNS server

RABBITMQ_PASS="your_rabbitmq_password" # Strong password for RabbitMQ
KEYSTONE_DBPASS="your_keystone_db_password" # Strong password for Keystone DB
ADMIN_PASS="your_admin_password"         # Strong password for OpenStack Admin user
DEMO_PASS="your_demo_password"           # Strong password for OpenStack Demo user
GLANCE_DBPASS="your_glance_db_password" # Strong password for Glance DB
GLANCE_PASS="your_glance_user_password" # Strong password for Glance user
NOVA_DBPASS="your_nova_db_password"     # Strong password for Nova DB
NOVA_PASS="your_nova_user_password"     # Strong password for Nova user
NEUTRON_DBPASS="your_neutron_db_password" # Strong password for Neutron DB
NEUTRON_PASS="your_neutron_user_password" # Strong password for Neutron user
METADATA_SECRET="your_metadata_secret_password" # Strong password for Metadata Agent

echo "--- Starting OpenStack Controller Node Installation ---"

# --- 1. System Update and Basic Configuration ---
echo "1. Performing system update and basic configuration..."
sudo apt update -y
sudo apt upgrade -y

echo "Setting hostname to ${CONTROLLER_HOSTNAME}..."
sudo hostnamectl set-hostname ${CONTROLLER_HOSTNAME}

echo "Configuring /etc/hosts file..."
sudo tee /etc/hosts > /dev/null <<EOF
127.0.0.1       localhost
${CONTROLLER_MGMT_IP}   ${CONTROLLER_HOSTNAME}
${COMPUTE1_MGMT_IP}     ${COMPUTE1_HOSTNAME}
${COMPUTE2_MGMT_IP}     ${COMPUTE2_HOSTNAME}
EOF

echo "Configuring network interfaces with static IPs (via Netplan)..."
sudo tee /etc/netplan/01-netcfg.yaml > /dev/null <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ${MGMT_INTERFACE}:
      dhcp4: no
      addresses: [${CONTROLLER_MGMT_IP}/24]
      gateway4: ${DEFAULT_GATEWAY}
      nameservers:
        addresses: [${DNS_SERVER}]
    ${DATA_INTERFACE}:
      dhcp4: no
      addresses: [192.168.1.10/24] # Adjust if your data network uses a different range
EOF
sudo netplan apply
echo "Network configuration applied. Please verify connectivity."
sleep 5 # Give network time to settle

echo "Installing and configuring NTP (chrony)..."
sudo apt install chrony -y
sudo sed -i '/^pool/c\pool 0.ubuntu.pool.ntp.org iburst\npool 1.ubuntu.pool.ntp.org iburst' /etc/chrony/chrony.conf
sudo systemctl restart chrony
sudo systemctl enable chrony
echo "NTP service configured."

# --- 2. Database Installation (MariaDB) ---
echo "2. Installing MariaDB database..."
sudo apt install mariadb-server python3-pymysql -y

echo "Configuring MariaDB..."
sudo sed -i "/^\[mysqld\]/a bind-address = ${CONTROLLER_MGMT_IP}\ndefault-storage-engine = innodb\ninnodb_file_per_table = on\nmax_connections = 4096\ncollation-server = utf8_general_ci\ncharacter-set-server = utf8" /etc/mysql/mariadb.conf.d/50-server.cnf
sudo systemctl restart mariadb
sudo systemctl enable mariadb
echo "MariaDB configured and restarted."

echo "Securing MariaDB installation (manual step recommended for setting root password):"
echo "Please run 'sudo mysql_secure_installation' manually after the script finishes."
# Or add expect script for automation, but interactive is safer for first time.

# --- 3. Message Queue Installation (RabbitMQ) ---
echo "3. Installing RabbitMQ message queue..."
sudo apt install rabbitmq-server -y

echo "Creating RabbitMQ user 'openstack'..."
sudo rabbitmqctl add_user openstack ${RABBITMQ_PASS}
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
echo "RabbitMQ user created and permissions set."
sudo systemctl restart rabbitmq-server
sudo systemctl enable rabbitmq-server

# --- 4. Identity Service Installation (Keystone) ---
echo "4. Installing Keystone Identity Service..."

echo "Creating Keystone database..."
sudo mysql -u root -p${MARIADB_ROOT_PASS} <<MYSQL_SCRIPT
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '${KEYSTONE_DBPASS}';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '${KEYSTONE_DBPASS}';
FLUSH PRIVILEGES;
MYSQL_SCRIPT
if [ $? -ne 0 ]; then echo "Keystone DB creation failed. Exiting."; exit 1; fi

echo "Installing Keystone package..."
sudo apt install keystone -y

echo "Configuring Keystone..."
sudo sed -i "/^\[database\]/a connection = mysql+pymysql://keystone:${KEYSTONE_DBPASS}@${CONTROLLER_HOSTNAME}/keystone" /etc/keystone/keystone.conf
sudo sed -i "/^\[token\]/a provider = fernet" /etc/keystone/keystone.conf

echo "Syncing Keystone database..."
sudo su -s /bin/sh -c "keystone-manage db sync" keystone
if [ $? -ne 0 ]; then echo "Keystone DB sync failed. Exiting."; exit 1; fi

echo "Setting up Fernet and Credential keys..."
sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

echo "Restarting Apache and enabling Keystone..."
sudo systemctl restart apache2
sudo systemctl enable apache2
echo "Keystone installed and configured."

# --- 5. Environment Variables Setup & User/Project Creation ---
echo "5. Setting up environment variables and creating OpenStack users/projects..."
echo "Setting up admin-openrc file..."
cat <<EOF > ~/admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=${ADMIN_PASS}
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://${CONTROLLER_HOSTNAME}:5000/v3
export OS_IDENTITY_API_VERSION=3
EOF
chmod +x ~/admin-openrc
source ~/admin-openrc

echo "Creating OpenStack users, projects, and roles..."
openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "Admin Project" admin
openstack user create --domain default --password ${ADMIN_PASS} admin
openstack role create admin
openstack grant role admin --project admin --user admin

openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" demo
openstack user create --domain default --password ${DEMO_PASS} demo
openstack grant role member --project demo --user demo
echo "Users, projects, and roles created."

# --- 6. Image Service Installation (Glance) ---
echo "6. Installing Glance Image Service..."

echo "Creating Glance database..."
sudo mysql -u root -p${MARIADB_ROOT_PASS} <<MYSQL_SCRIPT
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '${GLANCE_DBPASS}';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '${GLANCE_DBPASS}';
FLUSH PRIVILEGES;
MYSQL_SCRIPT
if [ $? -ne 0 ]; then echo "Glance DB creation failed. Exiting."; exit 1; fi

echo "Creating Glance user..."
openstack user create --domain default --password ${GLANCE_PASS} glance
openstack role add --project service --user glance admin

echo "Installing Glance packages..."
sudo apt install glance -y

echo "Configuring Glance..."
# glance-api.conf
sudo sed -i "/^\[database\]/a connection = mysql+pymysql://glance:${GLANCE_DBPASS}@${CONTROLLER_HOSTNAME}/glance" /etc/glance/glance-api.conf
sudo sed -i "/^\[glance_store\]/a stores = file,http\ndefault_store = file\nfilesystem_store_datadir = /var/lib/glance/images/" /etc/glance/glance-api.conf
sudo sed -i "/^\[keystone_authtoken\]/,/^\[/ s/^#.*//; /^\[keystone_authtoken\]/a www_authenticate_uri = http://${CONTROLLER_HOSTNAME}:5000\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nmemcached_servers = ${CONTROLLER_HOSTNAME}:11211\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nproject_name = service\nusername = glance\npassword = ${GLANCE_PASS}" /etc/glance/glance-api.conf

# glance-registry.conf
sudo sed -i "/^\[database\]/a connection = mysql+pymysql://glance:${GLANCE_DBPASS}@${CONTROLLER_HOSTNAME}/glance" /etc/glance/glance-registry.conf
sudo sed -i "/^\[keystone_authtoken\]/,/^\[/ s/^#.*//; /^\[keystone_authtoken\]/a www_authenticate_uri = http://${CONTROLLER_HOSTNAME}:5000\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nmemcached_servers = ${CONTROLLER_HOSTNAME}:11211\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nproject_name = service\nusername = glance\npassword = ${GLANCE_PASS}" /etc/glance/glance-registry.conf

echo "Syncing Glance database..."
sudo su -s /bin/sh -c "glance-manage db sync" glance
if [ $? -ne 0 ]; then echo "Glance DB sync failed. Exiting."; exit 1; fi

echo "Restarting Glance services..."
sudo systemctl restart glance-api glance-registry
sudo systemctl enable glance-api glance-registry
echo "Glance installed and configured."

echo "Uploading test image (Ubuntu 22.04 Cloud Image)..."
wget -P /tmp http://cloud-images.ubuntu.com/releases/jammy/release/ubuntu-22.04-server-cloudimg-amd64.img
openstack image create "Ubuntu 22.04" \
  --file /tmp/ubuntu-22.04-server-cloudimg-amd64.img \
  --disk-format qcow2 --container-format bare \
  --public
rm /tmp/ubuntu-22.04-server-cloudimg-amd64.img
echo "Test image uploaded."

# --- 7. Compute Service Installation (Nova - Controller Part) ---
echo "7. Installing Nova Compute Service (Controller Part)..."

echo "Creating Nova databases..."
sudo mysql -u root -p${MARIADB_ROOT_PASS} <<MYSQL_SCRIPT
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '${NOVA_DBPASS}';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '${NOVA_DBPASS}';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '${NOVA_DBPASS}';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '${NOVA_DBPASS}';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY '${NOVA_DBPASS}';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '${NOVA_DBPASS}';
FLUSH PRIVILEGES;
MYSQL_SCRIPT
if [ $? -ne 0 ]; then echo "Nova DB creation failed. Exiting."; exit 1; fi

echo "Creating Nova user..."
openstack user create --domain default --password ${NOVA_PASS} nova
openstack role add --project service --user nova admin

echo "Installing Nova controller packages..."
sudo apt install nova-api nova-conductor nova-scheduler nova-novncproxy -y

echo "Configuring Nova..."
# /etc/nova/nova.conf
sudo sed -i "/^\[api_database\]/a connection = mysql+pymysql://nova:${NOVA_DBPASS}@${CONTROLLER_HOSTNAME}/nova_api" /etc/nova/nova.conf
sudo sed -i "/^\[database\]/a connection = mysql+pymysql://nova:${NOVA_DBPASS}@${CONTROLLER_HOSTNAME}/nova" /etc/nova/nova.conf
sudo sed -i "/^\[DEFAULT\]/a transport_url = rabbit://openstack:${RABBITMQ_PASS}@${CONTROLLER_HOSTNAME}\nmy_ip = ${CONTROLLER_MGMT_IP}\nuse_neutron = True\nfirewall_driver = nova.virt.firewall.NoopFirewallDriver" /etc/nova/nova.conf
sudo sed -i "/^\[api\]/a auth_strategy = keystone" /etc/nova/nova.conf
sudo sed -i "/^\[keystone_authtoken\]/,/^\[/ s/^#.*//; /^\[keystone_authtoken\]/a www_authenticate_uri = http://${CONTROLLER_HOSTNAME}:5000\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nmemcached_servers = ${CONTROLLER_HOSTNAME}:11211\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nproject_name = service\nusername = nova\npassword = ${NOVA_PASS}" /etc/nova/nova.conf
sudo sed -i "/^\[vnc\]/a enabled = True\nserver_listen = \$my_ip\nserver_proxyclient_address = \$my_ip\nnovncproxy_base_url = http://${CONTROLLER_HOSTNAME}:6080/vnc_auto.html" /etc/nova/nova.conf
sudo sed -i "/^\[glance\]/a api_servers = http://${CONTROLLER_HOSTNAME}:9292" /etc/nova/nova.conf
sudo sed -i "/^\[oslo_concurrency\]/a lock_path = /var/lib/nova/tmp" /etc/nova/nova.conf


echo "Syncing Nova databases..."
sudo su -s /bin/sh -c "nova-manage api_db sync" nova
if [ $? -ne 0 ]; then echo "Nova API DB sync failed. Exiting."; exit 1; fi
sudo su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
if [ $? -ne 0 ]; then echo "Nova Cell0 mapping failed. Exiting."; exit 1; fi
sudo su -s /bin/sh -c "nova-manage db sync" nova
if [ $? -ne 0 ]; then echo "Nova DB sync failed. Exiting."; exit 1; fi
sudo su -s /bin/sh -c "nova-manage cell_v2 create_cell --name cell1 --verbose" nova
if [ $? -ne 0 ]; then echo "Nova Cell1 creation failed. Exiting."; exit 1; fi

echo "Restarting Nova controller services..."
sudo systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy
sudo systemctl enable nova-api nova-scheduler nova-conductor nova-novncproxy
echo "Nova controller services installed and configured."

# --- 8. Networking Service Installation (Neutron - Controller Part) ---
echo "8. Installing Neutron Networking Service (Controller Part)..."

echo "Creating Neutron database..."
sudo mysql -u root -p${MARIADB_ROOT_PASS} <<MYSQL_SCRIPT
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '${NEUTRON_DBPASS}';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '${NEUTRON_DBPASS}';
FLUSH PRIVILEGES;
MYSQL_SCRIPT
if [ $? -ne 0 ]; then echo "Neutron DB creation failed. Exiting."; exit 1; fi

echo "Creating Neutron user..."
openstack user create --domain default --password ${NEUTRON_PASS} neutron
openstack role add --project service --user neutron admin

echo "Installing Neutron controller packages..."
sudo apt install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent -y

echo "Configuring Neutron..."
# /etc/neutron/neutron.conf
sudo sed -i "/^\[database\]/a connection = mysql+pymysql://neutron:${NEUTRON_DBPASS}@${CONTROLLER_HOSTNAME}/neutron" /etc/neutron/neutron.conf
sudo sed -i "/^\[DEFAULT\]/a core_plugin = ml2\nservice_plugins = router\nallow_overlapping_ips = True\ntransport_url = rabbit://openstack:${RABBITMQ_PASS}@${CONTROLLER_HOSTNAME}\nauth_strategy = keystone\nnotify_nova_on_port_status_changes = True\nnotify_nova_on_security_group_changes = True" /etc/neutron/neutron.conf
sudo sed -i "/^\[keystone_authtoken\]/,/^\[/ s/^#.*//; /^\[keystone_authtoken\]/a www_authenticate_uri = http://${CONTROLLER_HOSTNAME}:5000\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nmemcached_servers = ${CONTROLLER_HOSTNAME}:11211\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nproject_name = service\nusername = neutron\npassword = ${NEUTRON_PASS}" /etc/neutron/neutron.conf
sudo sed -i "/^\[nova\]/a auth_url = http://${CONTROLLER_HOSTNAME}:5000\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nregion_name = RegionOne\nproject_name = service\nusername = nova\npassword = ${NOVA_PASS}" /etc/neutron/neutron.conf

# /etc/neutron/plugins/ml2/ml2_conf.ini
sudo sed -i "/^\[ml2\]/a type_drivers = flat,vlan,vxlan\ntenant_network_types = vxlan\nmechanism_drivers = linuxbridge,l2population\nextension_drivers = port_security" /etc/neutron/plugins/ml2/ml2_conf.ini
sudo sed -i "/^\[ml2_type_flat\]/a flat_networks = provider" /etc/neutron/plugins/ml2/ml2_conf.ini
sudo sed -i "/^\[ml2_type_vxlan\]/a vni_ranges = 1:1000" /etc/neutron/plugins/ml2/ml2_conf.ini
sudo sed -i "/^\[securitygroup\]/a enable_security_group = True\nenable_ipset = True\nfirewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver" /etc/neutron/plugins/ml2/ml2_conf.ini

# /etc/neutron/plugins/ml2/linuxbridge_agent.ini
sudo sed -i "/^\[linux_bridge\]/a physical_interface_mappings = provider:${DATA_INTERFACE}" /etc/neutron/plugins/ml2/linuxbridge_agent.ini
sudo sed -i "/^\[vxlan\]/a enable_vxlan = True\nlocal_ip = ${CONTROLLER_MGMT_IP}\nl2_population = True" /etc/neutron/plugins/ml2/linuxbridge_agent.ini
sudo sed -i "/^\[securitygroup\]/a enable_security_group = True\nfirewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver" /etc/neutron/plugins/ml2/linuxbridge_agent.ini

# /etc/neutron/l3_agent.ini
sudo sed -i "/^\[DEFAULT\]/a interface_driver = linuxbridge" /etc/neutron/l3_agent.ini

# /etc/neutron/dhcp_agent.ini
sudo sed -i "/^\[DEFAULT\]/a interface_driver = linuxbridge\ndhcp_driver = neutron.agent.linux.dhcp.Dnsmasq\nenable_isolated_metadata = True" /etc/neutron/dhcp_agent.ini

# /etc/neutron/metadata_agent.ini
sudo sed -i "/^\[DEFAULT\]/a nova_metadata_host = ${CONTROLLER_HOSTNAME}\nmetadata_proxy_shared_secret = ${METADATA_SECRET}" /etc/neutron/metadata_agent.ini

# Nova-Neutron Integration in nova.conf
sudo sed -i "/^\[neutron\]/a url = http://${CONTROLLER_HOSTNAME}:9696\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nregion_name = RegionOne\nproject_name = service\nusername = neutron\npassword = ${NEUTRON_PASS}\nservice_metadata_proxy = True\nmetadata_proxy_shared_secret = ${METADATA_SECRET}" /etc/nova/nova.conf

echo "Syncing Neutron database..."
sudo su -s /bin/sh -c "neutron-manage db sync" neutron
if [ $? -ne 0 ]; then echo "Neutron DB sync failed. Exiting."; exit 1; fi

echo "Restarting Neutron services..."
sudo systemctl restart neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent
sudo systemctl enable neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent
sudo systemctl restart nova-api # Restart Nova API for Neutron integration
echo "Neutron services installed and configured."

# --- 9. Dashboard Installation (Horizon) ---
echo "9. Installing Horizon Dashboard..."
sudo apt install openstack-dashboard -y

echo "Configuring Horizon..."
# /etc/openstack-dashboard/local_settings.py
sudo sed -i "s/^OPENSTACK_HOST = \"127.0.0.1\"/OPENSTACK_HOST = \"${CONTROLLER_HOSTNAME}\"/" /etc/openstack-dashboard/local_settings.py
sudo sed -i "s/^ALLOWED_HOSTS = \['localhost', '127.0.0.1'\]/ALLOWED_HOSTS = \['${CONTROLLER_HOSTNAME}', '${CONTROLLER_MGMT_IP}'\]/" /etc/openstack-dashboard/local_settings.py
sudo sed -i "s/^TIME_ZONE = \"UTC\"/TIME_ZONE = \"Asia\/Dhaka\"/" /etc/openstack-dashboard/local_settings.py
# Add a strong SECRET_KEY if not already present or replace it
if ! grep -q "SECRET_KEY" /etc/openstack-dashboard/local_settings.py; then
    sudo sed -i "/^#\s*SECRET_KEY/a SECRET_KEY = 'a_very_long_and_random_string_of_characters_for_security'" /etc/openstack-dashboard/local_settings.py
fi
sudo sed -i "s/^OPENSTACK_KEYSTONE_URL = \"http:\/\/%s:5000\/v2.0\" % OPENSTACK_HOST/OPENSTACK_KEYSTONE_URL = \"http:\/\/%s:5000\/v3\" % OPENSTACK_HOST/" /etc/openstack-dashboard/local_settings.py
sudo sed -i "s/^#OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'/OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = \"Default\"/" /etc/openstack-dashboard/local_settings.py
sudo sed -i "/^OPENSTACK_API_VERSIONS = {/a \    \"identity\": 3," /etc/openstack-dashboard/local_settings.py

echo "Restarting Apache for Horizon..."
sudo systemctl reload apache2
echo "Horizon Dashboard installed and configured."

echo "--- OpenStack Controller Node Installation Complete! ---"
echo "Please verify services by running: source ~/admin-openrc && openstack service list"
echo "You can now access the dashboard at: http://${CONTROLLER_MGMT_IP}/ or http://${CONTROLLER_HOSTNAME}/"
echo "Remember to run 'sudo mysql_secure_installation' manually for MariaDB."
```

**স্ক্রিপ্ট ব্যবহারের পদ্ধতি:**

১.  উপরের কোডটি একটি ফাইলে সেভ করুন, যেমন `install_controller.sh`।
২.  ফাইলের শুরুতে থাকা `Configuration Variables` সেকশনের সব ভেরিয়েবল আপনার পরিবেশ অনুযায়ী আপডেট করুন। **প্রতিটি পাসওয়ার্ডকে একটি শক্তিশালী, অনন্য পাসওয়ার্ডে পরিবর্তন করুন।**
৩.  ফাইলটিকে এক্সিকিউটেবল পারমিশন দিন: `chmod +x install_controller.sh`।
৪.  `sudo ./install_controller.sh` কমান্ড ব্যবহার করে স্ক্রিপ্টটি চালান।
৫.  MariaDB ইনস্টলেশনের পর, এটি আপনাকে `sudo mysql_secure_installation` ম্যানুয়ালি চালানোর জন্য বলবে। এটি সম্পন্ন করুন।

-----

## ৩. কম্পিউট নোড ইনস্টলেশন স্ক্রিপ্ট (`install_compute.sh`)

এই স্ক্রিপ্টটি আপনার কম্পিউট নোডগুলোর (যেমন `compute1`, `compute2`) জন্য। এটি চালানোর আগে আপনাকে প্রতিটি কম্পিউট নোডের জন্য এর IP অ্যাড্রেস এবং হোস্টনেমসহ অন্যান্য বিবরণ আপডেট করতে হবে।

```bash
#!/bin/bash

# ---------------------------------------------------
# OpenStack Compute Node Installation Script
# Ubuntu LTS
#
# Before running:
# 1. Update variables in the 'Configuration Variables' section.
# 2. Ensure network interfaces are correctly named and configured.
# 3. Run with sudo/root privileges.
# ---------------------------------------------------

set -e # Exit immediately if a command exits with a non-zero status.

# --- Configuration Variables ---
# !!! IMPORTANT: Update these variables according to your specific compute node !!!
THIS_COMPUTE_HOSTNAME="compute1" # Hostname for THIS compute node (e.g., compute1 or compute2)
THIS_COMPUTE_MGMT_IP="10.0.0.11" # Management IP of THIS compute node (e.g., 10.0.0.11 or 10.0.0.12)

CONTROLLER_HOSTNAME="controller"
CONTROLLER_MGMT_IP="10.0.0.10"
COMPUTE1_HOSTNAME="compute1"
COMPUTE1_MGMT_IP="10.0.0.11"
COMPUTE2_HOSTNAME="compute2"
COMPUTE2_MGMT_IP="10.0.0.12"

MGMT_INTERFACE="enp0s3" # Management network interface name (e.g., enp0s3)
DATA_INTERFACE="enp0s8" # Data/Provider network interface name (e.g., enp0s8)
DEFAULT_GATEWAY="10.0.0.1" # Your network's default gateway
DNS_SERVER="8.8.8.8"     # Your preferred DNS server

RABBITMQ_PASS="your_rabbitmq_password" # Must match the one set on controller
NOVA_PASS="your_nova_user_password"     # Must match the one set on controller
NEUTRON_PASS="your_neutron_user_password" # Must match the one set on controller
METADATA_SECRET="your_metadata_secret_password" # Must match the one set on controller

# Determine virtualisation type (kvm if supported, else qemu)
VIRT_TYPE="qemu"
if [ $(egrep -c '(vmx|svm)' /proc/cpuinfo) -eq 1 ]; then
  VIRT_TYPE="kvm"
fi
echo "Detected virtualization type: ${VIRT_TYPE}"

echo "--- Starting OpenStack Compute Node Installation (${THIS_COMPUTE_HOSTNAME}) ---"

# --- 1. System Update and Basic Configuration ---
echo "1. Performing system update and basic configuration..."
sudo apt update -y
sudo apt upgrade -y

echo "Setting hostname to ${THIS_COMPUTE_HOSTNAME}..."
sudo hostnamectl set-hostname ${THIS_COMPUTE_HOSTNAME}

echo "Configuring /etc/hosts file..."
sudo tee /etc/hosts > /dev/null <<EOF
127.0.0.1       localhost
${CONTROLLER_MGMT_IP}   ${CONTROLLER_HOSTNAME}
${COMPUTE1_MGMT_IP}     ${COMPUTE1_HOSTNAME}
${COMPUTE2_MGMT_IP}     ${COMPUTE2_HOSTNAME}
EOF

echo "Configuring network interfaces with static IPs (via Netplan)..."
sudo tee /etc/netplan/01-netcfg.yaml > /dev/null <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    ${MGMT_INTERFACE}:
      dhcp4: no
      addresses: [${THIS_COMPUTE_MGMT_IP}/24]
      gateway4: ${DEFAULT_GATEWAY}
      nameservers:
        addresses: [${DNS_SERVER}]
    ${DATA_INTERFACE}:
      dhcp4: no
      addresses: [192.168.1.11/24] # Adjust if your data network uses a different range, e.g., 192.168.1.11 for compute1, 192.168.1.12 for compute2
EOF
sudo netplan apply
echo "Network configuration applied. Please verify connectivity."
sleep 5 # Give network time to settle

echo "Installing and configuring NTP (chrony)..."
sudo apt install chrony -y
sudo sed -i '/^pool/c\pool 0.ubuntu.pool.ntp.org iburst\npool 1.ubuntu.pool.ntp.org iburst' /etc/chrony/chrony.conf
sudo systemctl restart chrony
sudo systemctl enable chrony
echo "NTP service configured."

# --- 2. Compute Service Installation (Nova - Compute Part) ---
echo "2. Installing Nova Compute Service..."

echo "Installing Nova Compute packages..."
sudo apt install nova-compute -y

echo "Configuring Nova Compute..."
# /etc/nova/nova.conf
sudo sed -i "/^\[DEFAULT\]/a transport_url = rabbit://openstack:${RABBITMQ_PASS}@${CONTROLLER_HOSTNAME}\nmy_ip = ${THIS_COMPUTE_MGMT_IP}\nuse_neutron = True\nfirewall_driver = nova.virt.firewall.NoopFirewallDriver" /etc/nova/nova.conf
sudo sed -i "/^\[api\]/a auth_strategy = keystone" /etc/nova/nova.conf
sudo sed -i "/^\[keystone_authtoken\]/,/^\[/ s/^#.*//; /^\[keystone_authtoken\]/a www_authenticate_uri = http://${CONTROLLER_HOSTNAME}:5000\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nmemcached_servers = ${CONTROLLER_HOSTNAME}:11211\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nproject_name = service\nusername = nova\npassword = ${NOVA_PASS}" /etc/nova/nova.conf
sudo sed -i "/^\[vnc\]/a enabled = True\nserver_listen = 0.0.0.0\nserver_proxyclient_address = ${THIS_COMPUTE_MGMT_IP}\nnovncproxy_base_url = http://${CONTROLLER_HOSTNAME}:6080/vnc_auto.html" /etc/nova/nova.conf
sudo sed -i "/^\[glance\]/a api_servers = http://${CONTROLLER_HOSTNAME}:9292" /etc/nova/nova.conf
sudo sed -i "/^\[oslo_concurrency\]/a lock_path = /var/lib/nova/tmp" /etc/nova/nova.conf
sudo sed -i "/^\[libvirt\]/a virt_type = ${VIRT_TYPE}" /etc/nova/nova.conf
# Nova-Neutron Integration (copying from controller script for consistency, ensure it matches controller)
sudo sed -i "/^\[neutron\]/a url = http://${CONTROLLER_HOSTNAME}:9696\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nregion_name = RegionOne\nproject_name = service\nusername = neutron\npassword = ${NEUTRON_PASS}\nservice_metadata_proxy = True\nmetadata_proxy_shared_secret = ${METADATA_SECRET}" /etc/nova/nova.conf


echo "Restarting Nova Compute service..."
sudo systemctl restart nova-compute
sudo systemctl enable nova-compute
echo "Nova Compute installed and configured."

# --- 3. Networking Service Installation (Neutron - Linuxbridge Agent) ---
echo "3. Installing Neutron Linuxbridge Agent..."

echo "Installing Neutron Linuxbridge Agent packages..."
sudo apt install neutron-linuxbridge-agent -y

echo "Configuring Neutron Linuxbridge Agent..."
# /etc/neutron/neutron.conf (common settings for all neutron components)
sudo sed -i "/^\[DEFAULT\]/a transport_url = rabbit://openstack:${RABBITMQ_PASS}@${CONTROLLER_HOSTNAME}\nauth_strategy = keystone" /etc/neutron/neutron.conf
sudo sed -i "/^\[keystone_authtoken\]/,/^\[/ s/^#.*//; /^\[keystone_authtoken\]/a www_authenticate_uri = http://${CONTROLLER_HOSTNAME}:5000\nauth_url = http://${CONTROLLER_HOSTNAME}:5000\nmemcached_servers = ${CONTROLLER_HOSTNAME}:11211\nauth_type = password\nproject_domain_name = Default\nuser_domain_name = Default\nproject_name = service\nusername = neutron\npassword = ${NEUTRON_PASS}" /etc/neutron/neutron.conf

# /etc/neutron/plugins/ml2/linuxbridge_agent.ini
sudo sed -i "/^\[linux_bridge\]/a physical_interface_mappings = provider:${DATA_INTERFACE}" /etc/neutron/plugins/ml2/linuxbridge_agent.ini
sudo sed -i "/^\[vxlan\]/a enable_vxlan = True\nlocal_ip = ${THIS_COMPUTE_MGMT_IP}\nl2_population = True" /etc/neutron/plugins/ml2/linuxbridge_agent.ini
sudo sed -i "/^\[securitygroup\]/a enable_security_group = True\nfirewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver" /etc/neutron/plugins/ml2/linuxbridge_agent.ini

echo "Restarting Neutron Linuxbridge Agent..."
sudo systemctl restart neutron-linuxbridge-agent
sudo systemctl enable neutron-linuxbridge-agent
echo "Neutron Linuxbridge Agent installed and configured."

echo "--- OpenStack Compute Node Installation Complete (${THIS_COMPUTE_HOSTNAME}) ---"
echo "Please verify services on the controller: source ~/admin-openrc && openstack compute service list && openstack network agent list"
```

**স্ক্রিপ্ট ব্যবহারের পদ্ধতি:**

১.  উপরের কোডটি একটি ফাইলে সেভ করুন, যেমন `install_compute.sh`।
২.  ফাইলের শুরুতে থাকা `Configuration Variables` সেকশনের সব ভেরিয়েবল আপনার **বর্তমান কম্পিউট নোডের জন্য** আপডেট করুন। উদাহরণস্বরূপ, যদি আপনি `compute1` মেশিনে এটি চালান, তাহলে `THIS_COMPUTE_HOSTNAME` এবং `THIS_COMPUTE_MGMT_IP` সেই অনুযায়ী সেট করুন। পাসওয়ার্ডগুলো কন্ট্রোলার নোডে যা ব্যবহার করেছেন, ঠিক তাই ব্যবহার করুন।
৩.  ফাইলটিকে এক্সিকিউটেবল পারমিশন দিন: `chmod +x install_compute.sh`।
৪.  `sudo ./install_compute.sh` কমান্ড ব্যবহার করে স্ক্রিপ্টটি চালান।
৫.  `compute1` ইনস্টল করার পর, একই স্ক্রিপ্ট `compute2` এর জন্য চালান, কিন্তু `THIS_COMPUTE_HOSTNAME` এবং `THIS_COMPUTE_MGMT_IP` এর মান `compute2` এর জন্য পরিবর্তন করে নিন।

-----

## ইনস্টলেশন পরবর্তী যাচাইকরণ

সব স্ক্রিপ্ট সফলভাবে চলার পর, কন্ট্রোলার নোডে গিয়ে নিচের কমান্ডগুলো চালিয়ে নিশ্চিত করুন যে সবকিছু সঠিকভাবে কাজ করছে:

```bash
source ~/admin-openrc
openstack service list # নিশ্চিত করুন যে সব পরিষেবা 'Up' আছে
openstack compute service list # নিশ্চিত করুন যে আপনার compute নোডগুলো 'Up' এবং 'enabled' দেখাচ্ছে
openstack network agent list # নিশ্চিত করুন যে বিভিন্ন নিউট্রন এজেন্টগুলো 'Up' দেখাচ্ছে
```

যদি কোনো সমস্যা হয়, সংশ্লিষ্ট পরিষেবার লগ ফাইলগুলো (`/var/log/nova`, `/var/log/neutron`, `/var/log/glance`, `/var/log/apache2`) পরীক্ষা করুন।

এই স্ক্রিপ্টগুলো আপনাকে ম্যানুয়াল ইনস্টলেশনের জটিলতা অনেকটাই কমিয়ে দেবে, কিন্তু প্রতিটি লাইনের কার্যকারিতা বোঝা এবং আপনার নির্দিষ্ট পরিবেশের সাথে মানিয়ে নেওয়া অত্যাবশ্যক।
