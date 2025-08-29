
# Install OpenStack Manually 
---
Reference: 
1) https://www.youtube.com/playlist?list=PLVV1alynPj3E4s4Nt2VM2J5Q0gl69WDR7
---
#### Requirement:

We are using two nodes
- 1. Controller
- 2. Compute

Operating System
Ubuntu Latest Version

Change Hostname
```
sudo hostnamectl set-hostname cloud3
```
Configure Static IPs
```
vi /etc/netplan/50
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.0.87/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.106.15/24

```
netplan apply
```
```
service networking restart
```

Update /etc/hosts file
```
vi /etc/hosts
```
```
192.168.106.87 controller
#10.0.2.50 compute
```
Configure Password less authentication on both vms
```
ssh-keygen
```
#go to compute node
#change root password
```
passwd root
```
```
vi /etc/ssh/sshd_config
```
#uncomment
```
PermitRootLogin yes
```
```
service sshd restart
```
#copy ssh-key from controller to compute node
```
ssh-copy-id -i root@compute
```
#Install NTP
```
apt install chrony -y
```
```
vi /etc/chrony/chrony.conf
```
comment default ntp server
#server 0.asia.pool.ntp.org
#server 1.asia.pool.ntp.org
#server 2.asia.pool.ntp.org
#server 3.asia.pool.ntp.org

add own ntp server its host ip
```
server 192.168.0.89 iburst
allow 192.168.106.15/24
```
```
service chrony restart
```
#go to compute node ntp configure
```
vi /etc/chrony/chrony.conf
```
#comment default ntp server
#pool 0.ubuntu.pool.ntp.org
#pool 1
#pool 2

#add controller server ip as ntp server
```
server controller iburst
```
```
service chrony restart
```
Verify ntp server
```
chronyc sources
```
#Now install OpenStack
> openstack.org
> Docs
> Installation Guide
> choose version and
Following [Environment] Section step by step:

## 1st Create Environment for OpenStack Installation ##

Add OpenStack Packages on both node
```
add-apt-repository cloud-archive:epoxy
```
install openstackclient on both node
```
apt install python3-openstackclient -y
```
Install and configure SQL Database
```
apt install mariadb-server python3-pymysql -y
```
```
vi /etc/mysql/mariadb.conf.d/99-openstack.cnf

```
```
[mysqld]
bind-address = 192.168.106.15

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

```
service mysql restart
```
choose a suitable password [****] for the database root account

```
mysql_secure_installation
```
Type:
- Enter
- y
- y
- type password
- y
- y
- y
- y

Now Verifying the mysql 
```
mysql -u root -p
```
Type mysql password
Then enter mysql terminal like:
```
MariaDB [(none)]>

```
```
show databases;
```
```
exit;
```
Install and configure rabbitmq-server
```
apt install rabbitmq-server -y
```
```
service rabbitmq-server restart
```
```
service rabbitmq-server status
```
```
rabbitmqctl add_user openstack ubuntu
```
Permit configuration, write, and read access for the openstack user
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
Install and configure memcached
```
apt install memcached python3-memcache -y
```
edit /etc/memcached.conf file
```
vi /etc/memcached.conf
```
```
-l 192.168.106.15
```
```
service memcached restart
```
#### No need to install Etcd

### Now Following [ Install OpenStack services ] Option Step by Step ##

- go to Minimal deployment for your version as my zed

Install and configure keystone

Create Database for keystone

access mysql
```
mysql
```
Login mysql console show: MariaDB [(none)]> 
```
CREATE DATABASE keystone;
```
```
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'ubuntu';
EXIT;
```
exit from sql

Install keystone
```
apt install keystone -y
```
Edit the /etc/keystone/keystone.conf
```
vi /etc/keystone/keystone.conf
```
add new connection under [database] section and comment default connection.

[database]
```
connection = mysql+pymysql://keystone:ubuntu@controller/keystone
```
[token] section, configure the Fernet token provider.

[token]
```
provider = fernet
```
Populate the Identity service database.
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
Verifying in databases
```
mysql
```
```
show databases;
```
```
use keystone;
``` 
```
show tables;
```
```
exit;
```
Initialize Fernet key repositories.
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
Bootstrap the Identity service.
```
keystone-manage bootstrap --bootstrap-password ubuntu \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
 ``` 
  
Configure the Apache HTTP server

Edit the /etc/apache2/apache2.conf
```
vi /etc/apache2/apache2.conf
```
Add your ServerName under '''Global configuration''' section.
```
ServerName controller
```
```
service apache2 restart
```
Configure the administrative account by setting the proper environmental variables:
```
export OS_USERNAME=admin
export OS_PASSWORD=ubuntu
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```
Create a domain, projects, users, and roles
The Identity service provides authentication services for each OpenStack service. The authentication service uses a combination of domains, projects, users, and roles.

create a new domain would be:

```
openstack domain create --description "An Example Domain" example
```
Create the service project:
```
openstack project create --domain default \
  --description "Service Project" service
```  
Create the myproject project:
```
openstack project create --domain default \
  --description "Demo Project" myproject
```
Create the myuser user:
```
openstack user create --domain default \
  --password-prompt myuser
```
Create the myrole role:
```
openstack role create myrole
```
Add the myrole role to the myproject project and myuser user:
```
openstack role add --project myproject --user myuser myrole
```
Verify operation

Verify operation of the Identity service before installing other services.

 Note

Perform these commands on the controller node.

Unset the temporary OS_AUTH_URL and OS_PASSWORD environment variable:
```
unset OS_AUTH_URL OS_PASSWORD
```
As the admin user, request an authentication token:
```
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
```
 Note

This command uses the password for the admin user.

As the myuser user created in the previous, request an authentication token:
```
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name myproject --os-username myuser token issue

```
Create OpenStack client environment scripts

Create and edit the admin-openrc file and add the following content:

 Note

The OpenStack client also supports using a clouds.yaml file. For more information, see the os-client-config.
```
nano admin-openrc
```
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
Replace ADMIN_PASS with the password you chose for the admin user in the Identity service.

Create and edit the demo-openrc file and add the following content:
```
nano demo-openrc
```
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

Replace DEMO_PASS with the password you chose for the demo user in the Identity service.

Using the scripts

To run clients as a specific project and user, you can simply load the associated client environment script prior to running them. For example:

Load the admin-openrc file to populate environment variables with the location of the Identity service and the admin project and user credentials:
```
. admin-openrc
```
Request an authentication token:
```
openstack token issue
```
---

# OpenStack Glance Installation Guide (Ubuntu)  
*Step-by-step instructions to install and configure the OpenStack Image Service (Glance) on Ubuntu*

This guide walks you through installing and configuring the **Glance** service on the **controller node** in an OpenStack environment. Images are stored using the **local file system** for simplicity.

> ✅ **Supported Version**: OpenStack 2025.1 (Ubuntu)

---

## 🔧 Prerequisites

Before installing Glance, you must set up the database, create service credentials, and register API endpoints.

### 1. Create the Glance Database

Log in to your MariaDB/MySQL server as `root`:

```bash
mysql
```

Run the following SQL commands:

```sql
CREATE DATABASE glance;
```

Grant privileges to the `glance` database user (replace `GLANCE_DBPASS` with a secure password):

```sql
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
```

> 🔐 Example: Use `GLANCE_DBPASS=secretpassword123`

Exit the database client:

```sql
EXIT;
```

---

### 2. Source Admin Credentials

Load the admin credentials to gain access to administrative OpenStack commands:

```bash
. admin-openrc
```

> 💡 Ensure the `admin-openrc` file exists and contains correct OS credentials.

---

### 3. Create Glance User and Service

#### Create the `glance` user:
```bash
openstack user create --domain default --password-prompt glance
```

> Enter and confirm a strong password when prompted (e.g., `GLANCE_PASS`).

#### Add `admin` role to the `glance` user in the `service` project:
```bash
openstack role add --project service --user glance admin
```

> ⚠️ This command produces **no output** — success is silent.

#### Create the `glance` service entity:
```bash
openstack service create --name glance \
  --description "OpenStack Image" image
```

Expected Output:
```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

---

### 4. Create API Endpoints

Register public, internal, and admin endpoints for the Glance service:

```bash
openstack endpoint create --region RegionOne \
  image public http://controller:9292
```

```bash
openstack endpoint create --region RegionOne \
  image internal http://controller:9292
```

```bash
openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```

> ✅ All URLs point to `http://controller:9292` assuming your controller hostname is `controller`.

---

### 5. (Optional) Register Quota Limits in Keystone

If you want to enable **per-tenant image quotas**, register these limits:

```bash
openstack --os-cloud devstack-system-admin registered limit create \
  --service glance --default-limit 1000 --region RegionOne image_size_total
```

```bash
openstack --os-cloud devstack-system-admin registered limit create \
  --service glance --default-limit 1000 --region RegionOne image_stage_total
```

```bash
openstack --os-cloud devstack-system-admin registered limit create \
  --service glance --default-limit 100 --region RegionOne image_count_total
```

```bash
openstack --os-cloud devstack-system-admin registered limit create \
  --service glance --default-limit 100 --region RegionOne image_count_uploading
```

> ⚠️ You’ll need to enable `use_keystone_limits=True` later in the config.

---

## 📦 Install and Configure Glance Components

### 1. Install Glance Packages

On the controller node:

```bash
sudo apt update
sudo apt install glance -y
```

---

### 2. Configure `glance-api.conf`

Edit the main configuration file:

```bash
sudo nano /etc/glance/glance-api.conf
```

### 🔹 [database] – Configure Database Access

In the `[database]` section:

```ini
[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```

> 🔁 Replace `GLANCE_DBPASS` with the actual password used earlier.

---

### 🔹 [keystone_authtoken] – Configure Identity Authentication

> ❗ Clear any existing options in this section before adding these.

```ini
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS
```

> 🔁 Replace `GLANCE_PASS` with the password you set for the `glance` user.

---

### 🔹 [paste_deploy] – Set Paste Flavor

```ini
[paste_deploy]
flavor = keystone
```

---

### 🔹 [glance_store] – Configure Image Storage (Local File System)

Add or update the following sections:

```ini
[DEFAULT]
enabled_backends = fs:file
```

```ini
[glance_store]
default_backend = fs
```

```ini
[fs]
filesystem_store_datadir = /var/lib/glance/images/
```

> 📁 This sets the local directory where images will be stored.

---

### 🔹 [oslo_limit] – Enable Unified Limits (Optional)

Only if using per-tenant quotas:

```ini
[oslo_limit]
auth_url = http://controller:5000
auth_type = password
user_domain_id = default
username = glance
system_scope = all
password = GLANCE_PASS
endpoint_id = ENDPOINT_ID
region_name = RegionOne
```

> 🔁 Replace:
- `GLANCE_PASS`: Password for the `glance` user.
- `ENDPOINT_ID`: Use the **public image endpoint ID** created earlier.

To find the endpoint ID:

```bash
openstack endpoint list --service glance --region RegionOne
```

Look for the **public** endpoint and copy its ID (e.g., `340be3625e9b4239a6415d034e98aace`).

---

### 🔹 [DEFAULT] – Enable Quotas (Optional)

To enable quota enforcement:

```ini
[DEFAULT]
use_keystone_limits = True
```

> ⚠️ Only enable this if you registered the limits earlier.

---

### 6. Grant Reader Role for System Scope

Ensure the `glance` user can read system-scope resources like limits:

```bash
openstack role add --user glance --user-domain Default --system all reader
```

---

## 🗃️ Populate the Glance Database

Run the database synchronization:

```bash
sudo su -s /bin/sh -c "glance-manage db_sync" glance
```

> ⚠️ You may see deprecation warnings — these can be safely ignored.

---

## 🔄 Finalize Installation

Restart the Glance API service to apply all changes:

```bash
sudo service glance-api restart
```

> ✅ Glance is now ready to serve image requests.

---

## ✅ Verification Steps

After installation, verify Glance is working:

### 1. Source Admin Credentials Again

```bash
. admin-openrc
```

### 2. List Available Images (should be empty at first)

```bash
openstack image list
```

Expected Output:
```
+----+------+--------+------------------+--------+-------+
| ID | Name | Status | Server Type      | Schema | Size  |
+----+------+--------+------------------+--------+-------+
+----+------+--------+------------------+--------+-------+
```

If no errors occur, Glance is running correctly.

---

## 🛠 Troubleshooting Tips

| Issue | Solution |
|------|----------|
| `Connection refused` to database | Ensure MySQL is running and `glance` user has correct privileges |
| Authentication failures | Double-check `keystone_authtoken` settings and passwords |
| Service not starting | Check logs: `tail -f /var/log/glance/api.log` |
| Endpoint not found | Confirm endpoint creation with `openstack endpoint list --service image` |

---

## 📚 Next Steps

- Upload your first image:
  ```bash
  openstack image create "cirros" \
    --file cirros-0.5.2-x86_64-disk.img \
    --disk-format qcow2 --container-format bare --public
  ```
- Proceed to install **Nova (Compute Service)** or **Cinder (Block Storage)**.

---

## 📎 References

- Official Docs: [https://docs.openstack.org/glance/2025.1/install/install-ubuntu.html](https://docs.openstack.org/glance/2025.1/install/install-ubuntu.html)
- `oslo.limit` Configuration: [OpenStack oslo.limit Docs](https://docs.openstack.org/oslo.limit/latest/)

---

✅ **Congratulations!** You've successfully installed and configured the OpenStack Glance service on Ubuntu.
---

- Not yet Finish, Now Have a lot installation and configurations

To be Continue from bellow link:
https://docs.openstack.org/keystone/yoga/install/keystone-install-ubuntu.html
