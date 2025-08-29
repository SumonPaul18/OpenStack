
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
vi admin-openrc
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
vi demo-openrc
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

Grant privileges to the `glance` database user (replace `ubuntu` with a secure password):

```sql
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'ubuntu';
```

> 🔐 Example: Use `ubuntu=secretpassword123`

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

> Enter and confirm a strong password when prompted (e.g., `ubuntu`).

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

Verifying openstack endpoint list:
```
openstack endpoint list
```

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
sudo vi /etc/glance/glance-api.conf
```

### 🔹 [database] – Configure Database Access

In the `[database]` section:

```ini
[database]
connection = mysql+pymysql://glance:ubuntu@controller/glance
```

> 🔁 Replace `ubuntu` with the actual password used earlier.

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
password = ubuntu
```

> 🔁 Replace `ubuntu` with the password you set for the `glance` user.

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

To find the endpoint ID:

```bash
openstack endpoint list --service glance --region RegionOne
```
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

# OpenStack Glance Verification Guide (Ubuntu)  
*Step-by-step instructions to verify the Glance (Image Service) installation in OpenStack 2025.1*

After installing and configuring the **Glance** service on the controller node, it’s essential to **verify** that the service is running correctly and can manage images.

> ✅ **Supported Version**: OpenStack 2025.1 (Ubuntu)  
> 📌 This guide assumes Glance was installed using the [Ubuntu installation guide](https://docs.openstack.org/glance/2025.1/install/install-ubuntu.html).

---

## 🔍 Purpose of Verification

This guide helps you:
- Confirm the Glance service is up and reachable.
- Verify API endpoints are registered.
- Upload a test image.
- Validate that image operations work as expected.

---

## 🧰 Prerequisites

Before verifying Glance:
- The **Glance service must be installed and configured**.
- You must have access to the **controller node**.
- The `admin-openrc` file must be available with correct credentials.

---

## ✅ Step 1: Source Admin Credentials

Load administrative credentials to use OpenStack CLI commands:

```bash
. admin-openrc
```

> 💡 This sets environment variables like `OS_USERNAME`, `OS_PASSWORD`, etc.  
> Ensure the file exists and contains valid admin credentials.

---

## ✅ Step 2: Verify Glance Service Status

Check if the `glance-api` service is running:

```bash
sudo systemctl status glance-api
```

✅ **Expected Output**:
- `active (running)` status
- No recent errors in logs

🔧 If not running, start it:

```bash
sudo systemctl start glance-api
sudo systemctl enable glance-api
```

---

## ✅ Step 3: List Glance API Endpoints

Ensure the Image service endpoints were created correctly:

```bash
openstack endpoint list --service glance --interface public
openstack endpoint list --service glance --interface internal
openstack endpoint list --service glance --interface admin
```

✅ **Expected Output**:
- Three endpoints (`public`, `internal`, `admin`) pointing to `http://controller:9292`
- All with `enabled=True` and correct `RegionOne`

Example:
```
+----------------------------------+-----------+--------------+---------------------------+
| ID                               | Interface | Region       | URL                       |
+----------------------------------+-----------+--------------+---------------------------+
| 340be3625e9b4239a6415d034e98aace | public    | RegionOne    | http://controller:9292    |
| a6e4b153c2ae4c919eccfdbb7dceb5d2 | internal  | RegionOne    | http://controller:9292    |
| 0c37ed58103f4300a84ff125a539032d | admin     | RegionOne    | http://controller:9292    |
+----------------------------------+-----------+--------------+---------------------------+
```

---

## ✅ Step 4: Verify Glance Service Registration

Check that the `glance` service is registered in OpenStack:

```bash
openstack service list | grep image
```

✅ **Expected Output**:
```
| 8c2c7f1b9b5049ea9e63757b5533e6d2 | glance | image     |
```

> If missing, re-run the service creation command from the install guide.

---

## ✅ Step 5: List Available Images

Run the following command to list current images:

```bash
openstack image list
```

✅ **Expected Output**:
```
+----+------+--------+------------------+--------+-------+
| ID | Name | Status | Server Type      | Schema | Size  |
+----+------+--------+------------------+--------+-------+
+----+------+--------+------------------+--------+-------+
```

> 🔹 At this stage, the list should be **empty** — that’s normal.

> ❌ If you get an authentication or connection error, double-check:
- `keystone_authtoken` settings in `/etc/glance/glance-api.conf`
- Network connectivity to `controller:5000` (Keystone) and `controller:9292` (Glance)

---

## ✅ Step 6: Download and Upload a Test Image

Use a small test image (like **CirrOS**) to validate image upload and visibility.

### 1. Download CirrOS Image

```bash
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
```

> 🔁 You can use any version; update the URL accordingly.

---

### 2. Upload Image to Glance

```bash
openstack image create "cirros" \
  --file cirros-0.5.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

📌 **Parameters Explained**:
- `--file`: Path to the image file
- `--disk-format`: Disk format (`qcow2`, `raw`, `vmdk`, etc.)
- `--container-format`: Container type (`bare`, `ovf`, etc.)
- `--public`: Makes the image accessible to all projects

---

### 3. Confirm Image Upload

List images again:

```bash
openstack image list
```

✅ **Expected Output**:
```
+--------------------------------------+--------+--------+------------------+-------------+--------+
| ID                                   | Name   | Status | Server Type      | Schema      | Size   |
+--------------------------------------+--------+--------+------------------+-------------+--------+
| 6a51c3d7-4c32-48bd-9a65-85e9a13f8b34 | cirros | active | None             | None        | 12717056 |
+--------------------------------------+--------+--------+------------------+-------------+--------+
```

> 🔹 The image should be in **`active`** status.

---

## ✅ Step 7: View Image Details (Optional)

Get detailed info about the uploaded image:

```bash
openstack image show cirros
```

Sample Output:
```yaml
id: 6a51c3d7-4c32-48bd-9a65-85e9a13f8b34
name: cirros
status: active
disk_format: qcow2
container_format: bare
size: 12717056
visibility: public
```

---

## ✅ Step 8: Verify Image Storage Location (Optional)

If using **local file storage**, confirm the image is saved on disk:

```bash
ls -la /var/lib/glance/images/
```

✅ You should see a file matching the image `ID` (e.g., `6a51c3d7-4c32-48bd-9a65-85e9a13f8b34`).

> 🔹 This confirms Glance is writing images to the configured directory.

---

## 🛠 Troubleshooting Common Issues

| Problem | Possible Cause | Solution |
|-------|----------------|----------|
| `Unable to establish connection to http://controller:9292` | Glance service not running | Run: `sudo systemctl restart glance-api` |
| `HTTP 401 Unauthorized` | Incorrect `keystone_authtoken` config | Check password, username, and auth URL in `glance-api.conf` |
| Image stuck in `queued` or `saving` state | Permission issue on image directory | Ensure `/var/lib/glance/images/` is owned by `glance:glance` |
| Endpoint not found | Missing or incorrect endpoint | Recreate endpoints using `openstack endpoint create` |
| `No such file or directory` during upload | Image file not found | Confirm path and permissions on the `.img` file |

Check logs for details:

```bash
sudo tail -20 /var/log/glance/glance-api.log
```

---

## 🚀 Next Steps

Now that Glance is verified:
- Proceed to install **Nova (Compute Service)**.
- Launch your first VM using the uploaded CirrOS image.
- Test image sharing between projects (if using `shared` visibility).

---

## 📎 References

- Official Docs: [https://docs.openstack.org/glance/2025.1/install/verify.html](https://docs.openstack.org/glance/2025.1/install/verify.html)
- Glance CLI Guide: [OpenStack Image CLI](https://docs.openstack.org/python-openstackclient/latest/cli/command-objects/image.html)

---

✅ **Congratulations!** You’ve successfully verified the Glance installation. The Image Service is ready for production use.

---

# OpenStack Placement Service Installation Guide for Ubuntu

This guide provides **step-by-step instructions** to install and configure the **OpenStack Placement service** on **Ubuntu**. The Placement service tracks inventory and usage of resources (like compute, memory, and disk) in an OpenStack cloud.

> ✅ **Note**: This guide is based on the official OpenStack documentation for the 2025.1 release and tailored for Ubuntu systems.

---

## 🔧 Prerequisites

Before installing the Placement service, you must set up a database, create service credentials, and register API endpoints.

### Step 1: Create the Placement Database

1. Connect to the MariaDB/MySQL database as the `root` user:

   ```bash
   sudo mysql
   ```

2. Create the `placement` database:

   ```sql
   CREATE DATABASE placement;
   ```

3. Grant privileges to the `placement` database user:
   
   Replace `ubuntu` with a strong password.

   ```sql
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'ubuntu';
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'ubuntu';
   ```

4. Exit the database client:

   ```sql
   EXIT;
   ```

---

### Step 2: Create Service User and Endpoints

1. Source the `admin` credentials to get administrative access:

   ```bash
   . admin-openrc
   ```

   > Ensure the `admin-openrc` file exists and contains the correct admin credentials.

2. Create the `placement` user in OpenStack Identity (Keystone):

   ```bash
   openstack user create --domain default --password-prompt placement
   ```

   - When prompted, enter a password (e.g., `ubuntu`) and confirm it.

3. Add the `placement` user to the `service` project with the `admin` role:

   ```bash
   openstack role add --project service --user placement admin
   ```

   > This command produces no output on success.

4. Create the Placement service entry in the service catalog:

   ```bash
   openstack service create --name placement --description "Placement API" placement
   ```

   Example Output:
   ```
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | Placement API                    |
   | name        | placement                        |
   | type        | placement                        |
   +-------------+----------------------------------+
   ```

5. Create the Placement API endpoints (public, internal, admin):

   > Replace `controller` with your controller node’s hostname if different.

   ```bash
   openstack endpoint create --region RegionOne placement public http://controller:8778
   openstack endpoint create --region RegionOne placement internal http://controller:8778
   openstack endpoint create --region RegionOne placement admin http://controller:8778
   ```

   > 🔔 **Note**: The default port is `8778`. Adjust if your environment uses a different port (e.g., `8780`).

---

## 📦 Install and Configure Placement Components

### Step 3: Install the Placement Package

Install the Placement API package using APT:

```bash
sudo apt update
sudo apt install placement-api
```

---

### Step 4: Configure the Placement Service

Edit the main configuration file:

```bash
sudo nano /etc/placement/placement.conf
```

#### 1. Configure Database Access

In the `[placement_database]` section, set the database connection string:

```ini
[placement_database]
connection = mysql+pymysql://placement:ubuntu@controller/placement
```

> 🔁 Replace `ubuntu` with the password you set earlier.

#### 2. Configure API and Authentication

In the `[api]` section, ensure the auth strategy is set to Keystone:

```ini
[api]
auth_strategy = keystone
```

In the `[keystone_authtoken]` section, configure authentication settings:

```ini
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = ubuntu
```

> 🔁 Replace `ubuntu` with the password you assigned to the `placement` user.

> ⚠️ **Important**:
> - Comment out or remove any other existing options in `[keystone_authtoken]`.
> - Ensure domain names (`Default`) match your Keystone configuration (case-sensitive).

---

### Step 5: Sync the Placement Database

Populate the database with initial schema and data:

```bash
sudo su -s /bin/sh -c "placement-manage db sync" placement
```

> 🟡 You may see deprecation warnings — these can be safely ignored.

---

## 🔄 Finalize Installation

### Step 6: Restart the Web Server

The Placement API runs under Apache. Reload the service to apply changes:

```bash
sudo service apache2 restart
```

---

## ✅ Verification Steps

To verify that the Placement service is working:

1. List available services:

   ```bash
   openstack service list | grep placement
   ```

   Expected Output:
   ```
   | 2d1a27022e6e4185b86adac4444c495f | placement | placement     | Placement API |
   ```

2. List Placement API endpoints:

   ```bash
   openstack endpoint list | grep placement
   ```

3. Test API access (optional):

   ```bash
   curl -s http://controller:8778 | python3 -m json.tool
   ```

   You should see a JSON response listing available versions.

---

## 🛠 Troubleshooting Tips

| Issue | Solution |
|------|----------|
| `Unable to connect to database` | Verify MySQL host, user, password, and network access |
| `Authentication failed` | Double-check `keystone_authtoken` settings and password |
| `404 Not Found` on API endpoint | Ensure Apache is running and placement WSGI is configured |
| `placement-manage: command not found` | Confirm `placement-api` package is installed |

Check logs for errors:
```bash
sudo tail -f /var/log/placement/placement-api.log
sudo tail -f /var/log/apache2/error.log
```

---

## 📚 Summary

You have now successfully:

✅ Created the Placement database  
✅ Registered the Placement service and endpoints  
✅ Installed and configured the Placement API  
✅ Synced the database and restarted services  

The Placement service is now ready to support Compute (Nova) and other resource tracking services in your OpenStack environment.

---

📌 **Next Steps**:
- Proceed to install and configure the **Nova (Compute)** service.
- Ensure Nova is configured to use the Placement API for resource tracking.

🔗 **Official Docs**: [OpenStack Placement Installation Guide](https://docs.openstack.org/placement/2025.1/install/install-ubuntu.html)

--- 
# OpenStack Placement Service Verification Guide

After installing and configuring the OpenStack Placement service, it's essential to **verify that it is functioning correctly**. This guide provides clear, step-by-step instructions based on the official [OpenStack Placement Verification documentation](https://docs.openstack.org/placement/2025.1/install/verify.html) for the 2025.1 release.

---

## ✅ Objective

Verify the correct operation of the **Placement service** by:
- Running upgrade checks
- Installing the `osc-placement` CLI plugin
- Listing resource classes and traits via the API

---

## 🔐 Step 1: Source Admin Credentials

Before performing any verification steps, you must authenticate as an administrative user.

```bash
. admin-openrc
```

> 💡 Ensure the `admin-openrc` file exists and contains the correct environment variables (e.g., OS_USERNAME, OS_PASSWORD, etc.). If not available, use an equivalent method to source admin credentials.

---

## 🧪 Step 2: Run Placement Upgrade Check

This command verifies the database schema and checks for potential upgrade issues.

```bash
placement-status upgrade check
```

### ✅ Expected Output (Example):

```
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
```

### 🟡 Troubleshooting Tips:
- If you see errors like **"Unable to connect to database"**, verify:
  - Database host, username, password in `/etc/placement/placement.conf`
  - Network connectivity to the database server
- If authentication fails, double-check `[keystone_authtoken]` settings.

> 📝 **Note**: You may see deprecation warnings — these are safe to ignore during verification.

---

## 🔌 Step 3: Install `osc-placement` CLI Plugin

The `osc-placement` plugin enables OpenStack CLI commands to interact with the Placement API.

### Option A: Install via pip (Python Package Index)

```bash
pip3 install osc-placement
```

> ✅ Recommended if you're using a virtual environment or don't have distribution packages.

### Option B: Install via Ubuntu/Debian Package

```bash
sudo apt install python3-osc-placement
```

> ✅ Use this if you prefer system packages managed by APT.

### Verify Installation

Check if the plugin is loaded:

```bash
openstack help | grep -i placement
```

You should see new commands like:
- `resource class list`
- `trait list`
- `allocation list`, etc.

---

## 🔍 Step 4: Test Placement API – List Resource Classes

Resource classes represent types of resources tracked by Placement (e.g., disk, memory, CPU).

Run this command to list them:

```bash
openstack --os-placement-api-version 1.2 resource class list --sort-column name
```

> 🔁 The `--os-placement-api-version` flag ensures compatibility. Version `1.2` supports resource class listing.

### ✅ Sample Output:

```
+----------------------------+
| name                       |
+----------------------------+
| DISK_GB                    |
| IPV4_ADDRESS               |
| MEMORY_MB                  |
| VCPU                       |
| CUSTOM_FPGA_XILINX_VU9P    |
| ...                        |
+----------------------------+
```

> 🟡 If you get a 404 or connection error, ensure:
> - Apache is running: `sudo systemctl status apache2`
> - Endpoint URLs are correct: `openstack endpoint list --service placement`

---

## 🏷️ Step 5: Test Placement API – List Traits

Traits are metadata tags used to describe capabilities or properties of resource providers (e.g., `COMPUTE_HYPRTENSION_ENABLED`).

List all available traits:

```bash
openstack --os-placement-api-version 1.6 trait list --sort-column name
```

> 🔁 Version `1.6` introduces trait support in the API.

### ✅ Sample Output:

```
+---------------------------------------+
| name                                  |
+---------------------------------------+
| COMPUTE_DEVICE_TAGGING                |
| COMPUTE_NET_ATTACH_INTERFACE          |
| COMPUTE_VOLUME_MULTI_ATTACH           |
| HW_CPU_X86_SSE                        |
| CUSTOM_TRAIT_EXAMPLE                  |
| ...                                   |
+---------------------------------------+
```

> 🟢 Success means:
> - The Placement API is reachable
> - Authentication works
> - Database is synced and populated

---

## 🛠 Troubleshooting Common Issues

| Problem | Solution |
|-------|----------|
| `Command 'openstack' not found` | Install OpenStack client: `sudo apt install python3-openstackclient` |
| `HTTP 401 Unauthorized` | Check `keystone_authtoken` credentials in `/etc/placement/placement.conf` |
| `HTTP 404 Not Found` | Confirm endpoint URL (`http://controller:8778`) and Apache configuration |
| `placement-status: command not found` | Ensure `placement-common` package is installed |

### Check Logs for Errors

```bash
sudo tail -f /var/log/placement/placement-api.log
sudo tail -f /var/log/apache2/error.log
```

Look for:
- Database connection errors
- Keystone authentication failures
- WSGI application loading issues

---

## ✅ Summary: Verification Checklist

| Task | Status |
|------|--------|
| ☑️ Source admin credentials | ✅ |
| ☑️ Run `placement-status upgrade check` | ✅ |
| ☑️ Install `osc-placement` plugin | ✅ |
| ☑️ List resource classes | ✅ |
| ☑️ List traits | ✅ |
| ☑️ Confirm API accessibility | ✅ |

---

## 📚 Next Steps

Now that the Placement service is verified:
- Proceed to install and configure **Nova (Compute) Controller Services**
- Ensure Nova is configured to use the Placement API
- Later, verify integration using:  
  ```bash
  openstack hypervisor stats show
  ```

🔗 **Official Docs**:  
[Verify Placement Installation](https://docs.openstack.org/placement/2025.1/install/verify.html)

---


---
- Not yet Finish, Now Have a lot installation and configurations

To be Continue from bellow link:
https://docs.openstack.org/keystone/yoga/install/keystone-install-ubuntu.html
