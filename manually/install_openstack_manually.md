
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
sudo apt install placement-api -y
```

---

### Step 4: Configure the Placement Service

Edit the main configuration file:

```bash
sudo vi /etc/placement/placement.conf
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

# OpenStack Nova (Compute) Controller Node Installation Guide for Ubuntu  
**Simple & Step-by-Step Deployment Guide**  

> ✅ Based on: [OpenStack Nova Install Guide (2025.1)](https://docs.openstack.org/nova/2025.1/install/controller-install-ubuntu.html)  
> 🖥️ Role: **Controller Node**  
> 📦 Distribution: **Ubuntu**  
> 🔧 Focus: Clear, easy-to-follow instructions with explanations

---

## 🧩 Overview

This guide walks you through installing and configuring the **Nova (Compute) service** on the **controller node** in an OpenStack environment.

Nova manages virtual machines (VMs), including creation, scheduling, and lifecycle management.

🔧 You will:
- Set up databases
- Create service users and endpoints
- Install Nova components
- Configure `nova.conf`
- Sync databases and register cells
- Start services

> ⚠️ Prerequisites:
> - MySQL/MariaDB, RabbitMQ, Keystone (Identity), Glance (Image), and Placement services **must already be installed and running**.

---

## 🗄️ Step 2: Create Nova Databases

Connect to your database server and create three databases for Nova.

### 1. Log in to MariaDB/MySQL as root:

```bash
sudo mysql
```

### 2. Create databases and grant privileges:

```sql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'ubuntu';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'ubuntu';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'ubuntu';
```

🔁 Replace `ubuntu` with a strong password (e.g., `nova_db_secret`).

### 3. Exit the database:

```sql
EXIT;
```

---

## 👤 Step 3: Create Nova Service User and Endpoints

### 1. Create the `nova` user in Keystone

```bash
openstack user create --domain default --password-prompt nova
```

When prompted, enter a password (e.g., `ubuntu`) and confirm it.

✅ Example Output:
```
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| name                | nova                             |
| ...                 | ...                              |
+---------------------+----------------------------------+
```

### 2. Assign the `admin` role to the `nova` user

```bash
openstack role add --project service --user nova admin
```

> 🟡 No output means success.

### 3. Register the Nova service in the catalog

```bash
openstack service create --name nova --description "OpenStack Compute" compute
```

✅ Expected Output:
```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

### 4. Create API Endpoints

```bash
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

> 🔔 Port `8774/v2.1` is the default for Nova API. Ensure `controller` resolves correctly.

---

## 📦 Step 4: Install Nova Packages

Install required Nova components on the **controller node**:

```bash
sudo apt update
sudo apt install nova-api nova-conductor nova-novncproxy nova-scheduler
```

🛠️ Components installed:
- `nova-api`: REST API endpoint
- `nova-conductor`: Mediates DB interactions
- `nova-scheduler`: Decides where to run VMs
- `nova-novncproxy`: Provides VNC console access

---

## ⚙️ Step 5: Configure `nova.conf`

Edit the main Nova configuration file:

```bash
sudo vi /etc/nova/nova.conf
```

Add or modify the following sections:

### 1. Database Access

```ini
[api_database]
connection = mysql+pymysql://nova:ubuntu@controller/nova_api

[database]
connection = mysql+pymysql://nova:ubuntu@controller/nova
```

🔁 Replace `ubuntu` with the database password you set earlier.

---

### 2. RabbitMQ Message Queue

```ini
[DEFAULT]
transport_url = rabbit://openstack:ubuntu@controller:5672/
```

🔁 Replace `ubuntu` with the password for the `openstack` user in RabbitMQ.

---

### 3. Keystone Authentication

```ini
[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = ubuntu
```

🔁 Replace `ubuntu` with the password you chose for the `nova` user.

> ❗ **Important**: Comment out or remove any other lines in `[keystone_authtoken]`.

---

### 4. Service User Token (Optional but Recommended)

```ini
[service_user]
send_service_user_token = true
auth_url = http://controller:5000/v3
auth_strategy = keystone
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = ubuntu
```

🔁 Use same `ubuntu` as above.

---

### 5. Controller Node IP Address

```ini
[DEFAULT]
my_ip = 10.0.0.11
```

🔁 Replace `10.0.0.11` with the **management network IP** of your controller node.

---

### 6. VNC Configuration

```ini
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

This allows VNC console access via the dashboard.

---

### 7. Glance (Image Service) Access

```ini
[glance]
api_servers = http://controller:9292
```

Ensure Glance is reachable at port `9292`.

---

### 8. Lock Path for Concurrency

```ini
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

Create the directory if needed:

```bash
sudo mkdir -p /var/lib/nova/tmp
```

---

### 9. Placement Service Access

```ini
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

🔁 Replace `PLACEMENT_PASS` with the password you set for the `placement` user.

> ❗ Remove or comment out any other options in `[placement]`.

---

### ✅ Final Notes on Configuration
- An ellipsis (`...`) in config examples means keep existing defaults.
- Do **not** duplicate sections — edit existing ones or add if missing.
- Avoid mixing old and new configs.

---

## 🛠 Step 6: Populate and Initialize Nova Databases

Run these commands in order:

### 1. Sync the `nova-api` database

```bash
sudo su -s /bin/sh -c "nova-manage api_db sync" nova
```

> 🟡 Ignore deprecation warnings.

---

### 2. Register the `cell0` database

```bash
sudo su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

> `cell0` holds failed or deleted instances.

---

### 3. Create `cell1` (Primary Compute Cell)

```bash
sudo su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

✅ Sample Output:
```
Created cell with UUID: f690f4fd-2bc5-4f15-8145-db561a7b9d3d
```

---

### 4. Sync the main Nova database

```bash
sudo su -s /bin/sh -c "nova-manage db sync" nova
```

> This sets up schemas for `nova`, `nova_cell0`, and `nova_cell1`.

---

### 5. Verify Cells Are Registered

```bash
sudo su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

✅ Expected Output:
```
+-------+--------------------------------------+----------------------------+----------------------------------------------------+----------+
| Name  | UUID                                 | Transport URL              | Database Connection                                | Disabled |
+-------+--------------------------------------+----------------------------+----------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 | none:/                     | mysql+pymysql://nova:****@controller/nova_cell0    | False    |
| cell1 | f690f4fd-2bc5-4f15-8145-db561a7b9d3d | rabbit://openstack@...     | mysql+pymysql://nova:****@controller/nova_cell1    | False    |
+-------+--------------------------------------+----------------------------+----------------------------------------------------+----------+
```

> 🟢 Success means both `cell0` and `cell1` appear and are **not disabled**.

---

## 🔁 Step 7: Restart Nova Services

Apply all changes by restarting the Nova services:

```bash
sudo service nova-api restart
sudo service nova-scheduler restart
sudo service nova-conductor restart
sudo service nova-novncproxy restart
```

> ✅ All services should restart without errors.

---

## ✅ Step 8: Verify Installation

Now that everything is running, verify Nova works.

### 1. List OpenStack services

```bash
openstack compute service list
```

You should see:
- `nova-scheduler`
- `nova-conductor`
- `nova-compute` (once compute nodes are added)
- All services in `UP` state

### 2. Check cells communication (Optional Advanced Test)

```bash
sudo nova-manage cell_v2 list_hosts
```

Should show registered compute nodes.

---

## 🛠 Troubleshooting Tips

| Issue | Solution |
|------|----------|
| `nova-api` fails to start | Check `keystone_authtoken` settings and password |
| Database sync errors | Confirm DB connectivity and credentials in `nova.conf` |
| `cell_v2` command not found | Ensure `nova-conductor` package is installed |
| 503 Service Unavailable | Make sure Apache/Nginx is running; check `/var/log/nova/*.log` |
| Host not showing in `list_hosts` | Wait for compute node to register; check firewall and networking |

Check logs:
```bash
sudo tail -f /var/log/nova/nova-api.log
sudo tail -f /var/log/nova/nova-scheduler.log
```

---

## 📌 Summary Checklist

| Task | Status |
|------|--------|
| ☑️ Source admin credentials | ✅ |
| ☑️ Create `nova_api`, `nova`, `nova_cell0` DBs | ✅ |
| ☑️ Create `nova` user and endpoints | ✅ |
| ☑️ Install Nova packages | ✅ |
| ☑️ Configure `/etc/nova/nova.conf` | ✅ |
| ☑️ Sync databases and create cells | ✅ |
| ☑️ Restart services | ✅ |
| ☑️ Verify with `openstack compute service list` | ✅ |

---

## 🚀 Next Steps

After completing the controller setup:

1. ➡️ [Install Nova Compute Service](https://docs.openstack.org/nova/2025.1/install/compute-install-ubuntu.html) on **compute nodes**
2. ➡️ Install and configure **Neutron (Networking)**
3. ➡️ Launch your first instance using:
   ```bash
   openstack server create ...
   ```

---

🔗 **Official Docs**:  
[Nova Controller Installation (Ubuntu)](https://docs.openstack.org/nova/2025.1/install/controller-install-ubuntu.html)

🎯 You're now ready to manage compute resources in OpenStack!

---

---
# OpenStack Nova (Compute) Service Installation Guide for Ubuntu  
**Simple & Step-by-Step Guide for Compute Nodes**

> ✅ Based on: [OpenStack Nova Compute Install Guide (2025.1)](https://docs.openstack.org/nova/2025.1/install/compute-install-ubuntu.html)  
> 🖥️ Role: **Compute Node**  
> 📦 Distribution: **Ubuntu**  
> 🔧 Focus: Easy-to-follow, beginner-friendly instructions

---

## 🧩 Overview

This guide walks you through installing and configuring the **Nova Compute service** (`nova-compute`) on a **compute node** in your OpenStack environment.

The compute node runs virtual machines (VMs) using **KVM/QEMU** and connects to the controller for management.

🔧 You will:
- Install `nova-compute` package
- Configure `/etc/nova/nova.conf`
- Enable hardware acceleration (KVM) or fallback to QEMU
- Start the service
- Register the compute node from the controller

> ⚠️ Prerequisites:
> - Controller node must have **Keystone, Glance, Placement, and Nova (controller services)** already installed and working.
> - Network connectivity between controller and compute nodes.
> - NTP synchronized on all nodes.

---

## 📦 Step 1: Install Nova Compute Package

Log in to your **compute node** (e.g., `compute1`) and install the Nova compute service.

```bash
sudo apt update
sudo apt install nova-compute
```

🛠️ This installs:
- `nova-compute`: The main service that manages VMs
- Dependencies like `libvirt`, `qemu`, and `kvm`

---

## ⚙️ Step 2: Configure `nova.conf`

Edit the main Nova configuration file:

```bash
sudo nano /etc/nova/nova.conf
```

Update the following sections:

### 1. RabbitMQ Message Queue

In the `[DEFAULT]` section:

```ini
[DEFAULT]
transport_url = rabbit://openstack:ubuntu@controller
```

🔁 Replace `ubuntu` with the password you set for the `openstack` user in RabbitMQ.

> ✅ Example: If your RabbitMQ password is `rabbit_secret`, use:
> ```
> transport_url = rabbit://openstack:rabbit_secret@controller
> ```

---

### 2. Keystone Authentication

In the `[api]` and `[keystone_authtoken]` sections:

```ini
[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = ubuntu
```

🔁 Replace `ubuntu` with the password you chose for the `nova` user in Keystone.

> ❗ Remove or comment out any other lines in `[keystone_authtoken]`.

---

### 3. Service User Token (Optional but Recommended)

```ini
[service_user]
send_service_user_token = true
auth_url = http://controller:5000/v3
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = ubuntu
```

🔁 Use the same `ubuntu` as above.

---

### 4. Management IP Address

In the `[DEFAULT]` section:

```ini
[DEFAULT]
my_ip = 10.0.0.31
```

🔁 Replace `10.0.0.31` with the **management network IP address** of your compute node.

> ✅ Example: First compute node → `10.0.0.31`, second → `10.0.0.32`, etc.

---

### 5. VNC Console Access

In the `[vnc]` section:

```ini
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

📌 Explanation:
- `server_listen = 0.0.0.0`: Listens on all interfaces
- `novncproxy_base_url`: Where users access VM consoles via browser

> 🔁 If `controller` hostname is not resolvable from client machines, replace `controller` with its IP (e.g., `http://10.0.0.11:6080/vnc_auto.html`)

---

### 6. Glance (Image Service) Access

```ini
[glance]
api_servers = http://controller:9292
```

Ensure the Image service is reachable.

---

### 7. Lock Path for Concurrency

```ini
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

Create the directory if missing:

```bash
sudo mkdir -p /var/lib/nova/tmp
```

---

### 8. Placement Service Access

```ini
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = PLACEMENT_PASS
```

🔁 Replace `PLACEMENT_PASS` with the password you set for the `placement` user.

> ❗ Comment out or remove any other options in `[placement]`.

---

## 💡 Step 3: Check for Hardware Virtualization Support

Run this command to check if your CPU supports **hardware acceleration (KVM)**:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

### Interpret the Result:

| Output | Meaning | Action |
|-------|--------|--------|
| `1` or higher | ✅ KVM supported | No extra config needed |
| `0` | ❌ No KVM support | Configure Nova to use **QEMU** |

---

### If No Hardware Support (Output = 0): Use QEMU

Edit the libvirt configuration:

```bash
sudo nano /etc/nova/nova-compute.conf
```

Add or modify the `[libvirt]` section:

```ini
[libvirt]
virt_type = qemu
```

> 📝 This tells Nova to use software-based QEMU instead of hardware-accelerated KVM.

---

## 🔁 Step 4: Restart and Enable Nova Compute Service

Apply all changes:

```bash
sudo service nova-compute restart
```

Ensure it starts automatically on boot:

```bash
sudo systemctl enable nova-compute
```

---

## 🛠 Troubleshooting Tips

### Common Issue: `nova-compute` fails to start

Check the log:
```bash
sudo tail -f /var/log/nova/nova-compute.log
```

#### If you see:
```
AMQP server on controller:5672 is unreachable
```

✅ Fix:
- Ensure **RabbitMQ is running** on the controller.
- Open port `5672` on the controller’s firewall:

```bash
sudo ufw allow from 10.0.0.0/24 to any port 5672
```

Replace `10.0.0.0/24` with your management network.

Then restart:
```bash
sudo service nova-compute restart
```

---

## ➕ Step 5: Add Compute Node to Cell Database (On Controller)

> 🔧 This step must be done **on the controller node**, not the compute node.

### 1. Source Admin Credentials

```bash
. admin-openrc
```

### 2. Verify Compute Service is Running

```bash
openstack compute service list --service nova-compute
```

✅ Expected Output:
```
+----+-----------+--------------+------+---------+-------+----------------------------+
| ID | Host      | Binary       | Zone | Status  | State | Updated At                 |
+----+-----------+--------------+------+---------+-------+----------------------------+
|  1 | compute1  | nova-compute | nova | enabled | up    | 2025-04-05T10:00:00.000000 |
+----+-----------+--------------+------+---------+-------+----------------------------+
```

If state is `down`, check logs and network/firewall.

---

### 3. Discover Compute Hosts

Register the compute node(s) in the cell database:

```bash
sudo su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

✅ Sample Output:
```
Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting compute nodes from cell 'cell1': ad5a5985-a719-4567-98d8-8d148aaae4bc
Found 1 computes in cell: ad5a5985-a719-4567-98d8-8d148aaae4bc
Checking host mapping for compute host 'compute1': fe58ddc1-1d65-4f87-9456-bc040dc106b3
Creating host mapping for compute host 'compute1': fe58ddc1-1d65-4f87-9456-bc040dc106b3
```

> 🟢 Success means your compute node is now registered!

---

### 🔁 Optional: Automate Host Discovery

To avoid running `discover_hosts` manually every time you add a new compute node:

Edit `/etc/nova/nova.conf` on the **controller node**:

```ini
[scheduler]
discover_hosts_in_cells_interval = 300
```

This will automatically discover new compute nodes every 5 minutes.

Then restart services:

```bash
sudo service nova-scheduler restart
```

---

## ✅ Final Verification

Back on the **controller node**, verify everything works:

```bash
openstack compute service list
```

All services should be `up` and `enabled`.

Also run:

```bash
sudo nova-manage cell_v2 list_hosts
```

You should see your compute node listed under `cell1`.

---

## 📌 Summary Checklist

| Task | Status |
|------|--------|
| ☑️ Install `nova-compute` on compute node | ✅ |
| ☑️ Configure `/etc/nova/nova.conf` | ✅ |
| ☑️ Set correct `my_ip` | ✅ |
| ☑️ Enable KVM or set `virt_type = qemu` | ✅ |
| ☑️ Restart `nova-compute` service | ✅ |
| ☑️ Run `discover_hosts` on controller | ✅ |
| ☑️ Confirm host appears in `list_hosts` | ✅ |

---

## 🚀 Next Steps

Now that your compute node is online:

1. ➡️ Install and configure **Neutron (Networking)** on controller and compute nodes
2. ➡️ Upload an image to Glance (e.g., Ubuntu Cloud Image)
3. ➡️ Create networks, flavors, and launch your first VM!

Example:
```bash
openstack server create --image ubuntu2204 --flavor m1.small --network private-net --key-name mykey my-instance
```

---

🔗 **Official Docs**:  
[Nova Compute Installation (Ubuntu)](https://docs.openstack.org/nova/2025.1/install/compute-install-ubuntu.html)

🎯 You’re now ready to run virtual machines at scale!

---

# OpenStack Neutron (Networking) Service Installation Guide for Ubuntu  
**Simple & Step-by-Step Guide for Controller Node**

> ✅ Based on: [Neutron Controller Install Guide (2025.1)](https://docs.openstack.org/neutron/2025.1/install/controller-install-ubuntu.html)  
> 🖥️ Role: **Controller Node**  
> 📦 Distribution: **Ubuntu**  
> 🔧 Focus: Easy-to-follow, beginner-friendly instructions

---

## 🧩 Overview

This guide walks you through installing and configuring the **Neutron (Networking)** service on the **controller node** in your OpenStack environment.

Neutron provides networking capabilities for instances (VMs), including:
- Provider (external) networks
- Self-service (private) networks with routers and NAT
- DHCP, metadata, and security groups

🔧 You will:
- Create the Neutron database and service user
- Register API endpoints
- Install Neutron packages
- Configure core services and agents
- Integrate with Nova (Compute)
- Start and verify services

> ⚠️ Prerequisites:
> - MySQL/MariaDB, RabbitMQ, Keystone, Glance, Nova (controller & compute) services **must already be installed and working**
> - At least one compute node with `nova-compute` running

---

## 🗄️ Step 1: Create Neutron Database

Connect to your database server and create the `neutron` database.

### 1. Log in to MariaDB/MySQL as root:

```bash
mysql
```

> Enter the root password when prompted.

### 2. Create the `neutron` database:

```sql
CREATE DATABASE neutron;
```

### 3. Grant privileges:

```sql
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'ubuntu';
```

🔁 Replace `ubuntu` with a strong password (e.g., `neutron_db_secret`).

### 4. Exit the database:

```sql
EXIT;
```

---

## 👤 Step 3: Create Neutron Service User and Endpoints

Source Admin Credentials
Before starting, load admin credentials to use OpenStack CLI commands.
```
. admin-openrc
```

### 1. Create the `neutron` user in Keystone

```bash
openstack user create --domain default --password-prompt neutron
```

When prompted, enter a password (e.g., `ubuntu`) and confirm it.

✅ Example Output:
```
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| name                | neutron                          |
| ...                 | ...                              |
+---------------------+----------------------------------+
```

### 2. Assign the `admin` role to the `neutron` user

```bash
openstack role add --project service --user neutron admin
```

> 🟡 No output means success.

### 3. Register the Neutron service in the catalog

```bash
openstack service create --name neutron --description "OpenStack Networking" network
```

✅ Expected Output:
```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

### 4. Create API Endpoints

```bash
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

> 🔔 Port `9696` is the default for Neutron API.

---

## 📦 Step 4: Install Neutron Packages

Install required Neutron components on the **controller node**:

```bash
sudo apt update
sudo apt install neutron-server neutron-plugin-ml2 \
  neutron-openvswitch-agent neutron-l3-agent \
  neutron-dhcp-agent neutron-metadata-agent
```

🛠️ Components installed:
- `neutron-server`: REST API and core logic
- `ml2`: Modular Layer 2 plugin (supports VLAN, VXLAN, etc.)
- `openvswitch-agent`: OVS-based switching
- `l3-agent`: Routing and NAT
- `dhcp-agent`: DHCP service for networks
- `metadata-agent`: Provides metadata to instances

> 🔁 Note: We install all agents here, but only `neutron-server` runs on the controller. Other agents may also run on compute nodes depending on your architecture.

---

## ⚙️ Step 5: Configure ML2 (Modular Layer 2) Plugin

Edit the ML2 configuration file:

```bash
sudo nano /etc/neutron/plugins/ml2/ml2_conf.ini
```

Update the following sections:

### 1. Core Driver Types

```ini
[ml2]
type_drivers = flat,vlan,vxlan,gre
```

> ✅ For basic external networks: `flat,vlan`  
> ✅ For self-service networks: include `vxlan` or `gre`

---

### 2. Tenant Network Types

```ini
[ml2]
tenant_network_types = vxlan
```

> Use `vxlan` for overlay networks (recommended for self-service).  
> Use `vlan` for physical segmentation.

---

### 3. Mechanism Drivers

```ini
[ml2]
mechanism_drivers = openvswitch,l2population
```

- `openvswitch`: Uses OVS for switching
- `l2population`: Enables ARP responder and reduces broadcast traffic (only with `vxlan`/`gre`)

> ❌ If using `flat` or `vlan`, omit `l2population`.

---

### 4. Enable Port Security

```ini
[ml2]
extension_drivers = port_security
```

Allows use of security groups and port-level policies.

---

### 5. Configure Flat Networks (Optional)

If using flat provider networks:

```ini
[ml2_type_flat]
flat_networks = provider
```

Replace `provider` with your physical network name.

---

### 6. Configure VXLAN Self-Service Networks

```ini
[ml2_type_vxlan]
vni_ranges = 1:1000
```

Defines VXLAN VNI range for tenant networks.

---

### 7. OVS Agent Configuration

Ensure OVS agent uses tunneling (if using VXLAN):

```ini
[ovs]
bridge_mappings = provider:br-provider
local_ip = 10.0.0.11
```

- `bridge_mappings`: Maps physical network (`provider`) to OVS bridge (`br-provider`)
- `local_ip`: Management IP of controller node

---

### 8. Enable Tunneling (for VXLAN/GRE)

```ini
[agent]
tunnel_types = vxlan
```

---

## ⚙️ Step 6: Configure Neutron Server (`neutron.conf`)

Edit the main configuration file:

```bash
sudo nano /etc/neutron/neutron.conf
```

### 1. Database Access

```ini
[database]
connection = mysql+pymysql://neutron:ubuntu@controller/neutron
```

🔁 Replace `ubuntu` with the database password.

---

### 2. RabbitMQ Message Queue

```ini
[DEFAULT]
transport_url = rabbit://openstack:ubuntu@controller
```

🔁 Replace `ubuntu` with your RabbitMQ `openstack` user password.

---

### 3. Keystone Authentication

```ini
[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = ubuntu
```

🔁 Replace `ubuntu` with the password you set for the `neutron` user.

> ❗ Remove or comment out any other lines in `[keystone_authtoken]`.

---

### 4. Core Plugin

```ini
[DEFAULT]
core_plugin = ml2
service_plugins = router
```

- `ml2`: Enables ML2 plugin
- `router`: Enables L3 routing (required for self-service networks)

---

### 5. Lock Path

```ini
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

Create directory if needed:

```bash
sudo mkdir -p /var/lib/neutron/tmp
```

---

## 🌐 Step 7: Configure the Metadata Agent

The metadata agent delivers configuration (like SSH keys) to instances.

Edit:

```bash
sudo nano /etc/neutron/metadata_agent.ini
```

### In `[DEFAULT]` section:

```ini
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```

🔁 Replace `METADATA_SECRET` with a strong shared secret (e.g., `meta123secret`).

> This same secret must be used in Nova configuration.

---

## 🔗 Step 8: Configure Nova to Use Neutron

Neutron integrates with Nova to provide networking for instances.

Edit Nova config:

```bash
sudo nano /etc/nova/nova.conf
```

### Add or update the `[neutron]` section:

```ini
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = ubuntu
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```

🔁 Replace:
- `ubuntu` with the `neutron` user password
- `METADATA_SECRET` with the same value used in `metadata_agent.ini`

---

## 🔁 Step 9: Finalize Installation

### 1. Populate the Neutron Database

Run the database sync command:

```bash
sudo su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

> 🟡 Ignore deprecation warnings — this is normal.

---

### 2. Restart Nova API Service

Apply changes in Nova:

```bash
sudo service nova-api restart
```

---

### 3. Restart Neutron Services

Start all Neutron components:

```bash
sudo service neutron-server restart
sudo service neutron-openvswitch-agent restart
sudo service neutron-dhcp-agent restart
sudo service neutron-metadata-agent restart
sudo service neutron-l3-agent restart
```

> ✅ All services should start without errors.

Enable them on boot:

```bash
sudo systemctl enable neutron-server neutron-openvswitch-agent \
  neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

---

## ✅ Step 10: Verify Installation

### 1. Check Neutron Service List

```bash
openstack network agent list
```

✅ Expected Output:
```
+----+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+----+--------------------+------------+-------------------+-------+-------+---------------------------+
| 1  | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 2  | Open vSwitch agent | controller | None              | :-)   | UP    | neutron-openvswitch-agent |
| 3  | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| 4  | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
+----+--------------------+------------+-------------------+-------+-------+---------------------------+
```

All agents should show `UP`.

---

### 2. Confirm Integration with Nova

No errors should appear in logs:
```bash
sudo tail -f /var/log/nova/nova-api.log
sudo tail -f /var/log/neutron/*.log
```

---

## 🛠 Troubleshooting Tips

| Issue | Solution |
|------|----------|
| `neutron-server` fails to start | Check DB credentials, plugin settings, and file permissions |
| Agents show `DOWN` | Ensure RabbitMQ is accessible; open port `5672` |
| Instances can't get IP via DHCP | Verify `br-int` and `br-tun` bridges exist; restart DHCP agent |
| Metadata not working | Confirm `service_metadata_proxy = true` and shared secret match in Nova and Neutron |

---

## 📌 Summary Checklist

| Task | Status |
|------|--------|
| ☑️ Source admin credentials | ✅ |
| ☑️ Create `neutron` database | ✅ |
| ☑️ Create `neutron` user and endpoints | ✅ |
| ☑️ Install Neutron packages | ✅ |
| ☑️ Configure ML2 plugin | ✅ |
| ☑️ Configure `neutron.conf` | ✅ |
| ☑️ Set up metadata agent | ✅ |
| ☑️ Update Nova to use Neutron | ✅ |
| ☑️ Sync database and restart services | ✅ |
| ☑️ Verify with `openstack network agent list` | ✅ |

---

## 🚀 Next Steps

After Neutron is running:

1. ➡️ [Install Neutron on Compute Nodes](https://docs.openstack.org/neutron/2025.1/install/compute-install-ubuntu.html) (OVS agent only)
2. ➡️ Create provider or self-service networks
3. ➡️ Launch an instance and test connectivity

Example network:
```bash
openstack network create --share --external --provider-physical-network provider --provider-network-type flat provider-net
```

---

🔗 **Official Docs**:  
[Neutron Controller Installation (Ubuntu)](https://docs.openstack.org/neutron/2025.1/install/controller-install-ubuntu.html)


🎯 You're now ready to connect virtual machines to powerful, scalable networks!

---
# OpenStack Neutron (Networking) Service Installation Guide for Ubuntu  
**Simple & Step-by-Step Guide for Compute Nodes**

> ✅ Based on: [Neutron Compute Install Guide (2025.1)](https://docs.openstack.org/neutron/2025.1/install/compute-install-ubuntu.html)  
> 🖥️ Role: **Compute Node**  
> 📦 Distribution: **Ubuntu**  
> 🔧 Focus: Easy-to-follow, beginner-friendly instructions

---

## 🧩 Overview

This guide walks you through installing and configuring the **Neutron Networking service** on a **compute node** in your OpenStack environment.

The compute node uses Neutron to provide:
- Network connectivity for instances (VMs)
- Security groups (firewall rules)
- DHCP services
- Integration with OVS (Open vSwitch) for virtual switching

🔧 You will:
- Install the `neutron-openvswitch-agent`
- Configure `neutron.conf`
- Integrate Neutron with Nova
- Restart services

> ⚠️ Prerequisites:
> - Controller node must have **Neutron (controller services)** already installed and running
> - **Nova (Compute)** service must be installed on this node
> - RabbitMQ, Keystone, and networking connectivity must be working
> - You must have chosen a networking option (Option 1: Provider networks or Option 2: Self-service + Provider networks)

---

## 📦 Step 1: Install Neutron Components

Log in to your **compute node** and install the required Neutron agent.

```bash
$ sudo apt update
$ sudo apt install neutron-openvswitch-agent
```

🛠️ This installs:
- `neutron-openvswitch-agent`: Manages virtual networks using Open vSwitch (OVS)
- Dependencies like `openvswitch-switch`

> 💡 Note: Compute nodes do **not** run the full Neutron server (`neutron-server`), only the OVS agent.

---

## ⚙️ Step 2: Configure Common Neutron Settings

Edit the main Neutron configuration file:

```bash
$ sudo nano /etc/neutron/neutron.conf
```

Update the following sections:

### 1. Disable Database Access

Compute nodes do **not** access the database directly.

✅ Comment out any `connection` line in `[database]`:

```ini
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
# Comment out or remove any database connection lines
```

---

### 2. RabbitMQ Message Queue

In the `[DEFAULT]` section:

```ini
[DEFAULT]
transport_url = rabbit://openstack:ubuntu@controller
```

🔁 Replace `ubuntu` with the password for the `openstack` user in RabbitMQ.

> ✅ Example: If your RabbitMQ password is `rabbit_secret`, use:
> ```
> transport_url = rabbit://openstack:rabbit_secret@controller
> ```

---

### 3. Lock Path for Concurrency

```ini
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

Create the directory if needed:

```bash
$ sudo mkdir -p /var/lib/neutron/tmp
```

---

## 🔗 Step 3: Configure Networking Option (Choose One)

You must choose the **same networking option** that was configured on the **controller node**.

---

### 🔹 Option 1: Provider Networks (Simple External Networks)

Use this if you only want instances connected directly to external (flat or VLAN) networks.

#### Install and Configure OVS Components

```bash
$ sudo apt install openvswitch-switch openvswitch-common
```

#### Start and Enable OVS

```bash
$ sudo systemctl enable openvswitch-switch
$ sudo systemctl start openvswitch-switch
```

#### Configure OVS Agent

Edit:
```bash
$ sudo nano /etc/neutron/plugins/ml2/openvswitch_agent.ini
```

Add:
```ini
[ovs]
bridge_mappings = provider:br-provider

[agent]
tunnel_types =
```

> - `bridge_mappings`: Maps the physical network `provider` to OVS bridge `br-provider`
> - `tunnel_types = `: Empty because no overlay (VXLAN/GRE) tunnels are used

---

### 🔹 Option 2: Self-Service Networks (With VXLAN/GRE Overlay)

Use this to support private tenant networks with routing and NAT.

#### Install Required Packages

```bash
$ sudo apt install openvswitch-switch openvswitch-common
```

#### Configure OVS Agent

Edit:
```bash
$ sudo nano /etc/neutron/plugins/ml2/openvswitch_agent.ini
```

Add:
```ini
[ovs]
local_ip = 10.0.0.31
bridge_mappings = provider:br-provider

[agent]
tunnel_types = vxlan
l2_population = true
```

🔁 Replace `10.0.0.31` with the **management IP** of your compute node.

> - `local_ip`: Used for VXLAN tunnel endpoints
> - `tunnel_types = vxlan`: Enables VXLAN overlay networks
> - `l2_population`: Reduces flooding with ARP responder (recommended)

---

## 🔗 Step 4: Configure Nova to Use Neutron

Neutron integrates with Nova to manage network interfaces for VMs.

Edit Nova's config file:

```bash
$ sudo nano /etc/nova/nova.conf
```

### In the `[neutron]` section:

```ini
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = ubuntu
```

🔁 Replace `ubuntu` with the password you set for the `neutron` user in Keystone.

> ❗ If the `[neutron]` section doesn’t exist, create it.

---

## 🔁 Step 5: Finalize Installation

### 1. Restart Nova Compute Service

Apply the Neutron integration:

```bash
$ sudo service nova-compute restart
```

---

### 2. Restart Neutron OVS Agent

Start the networking agent:

```bash
$ sudo service neutron-openvswitch-agent restart
```

Enable it on boot:

```bash
$ sudo systemctl enable neutron-openvswitch-agent
```

---

## ✅ Step 6: Verify Installation (On Controller Node)

Now go to your **controller node** to verify that the compute node is recognized.

### 1. Source Admin Credentials

```bash
$ . admin-openrc
```

### 2. List Neutron Agents

```bash
$ openstack network agent list
```

✅ Expected Output:
```
+----+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+----+--------------------+------------+-------------------+-------+-------+---------------------------+
| 1  | Open vSwitch agent | compute1   | None              | :-)   | UP    | neutron-openvswitch-agent |
| 2  | Open vSwitch agent | controller | None              | :-)   | UP    | neutron-openvswitch-agent |
+----+--------------------+------------+-------------------+-------+-------+---------------------------+
```

> 🟢 Success if:
> - The agent for your compute node appears
> - `Alive = :-)`, `State = UP`

---

## 🛠 Troubleshooting Tips

| Issue | Solution |
|------|----------|
| `neutron-openvswitch-agent` fails to start | Check `/var/log/neutron/openvswitch-agent.log` |
| Agent shows `DOWN` or `XXX` | Ensure RabbitMQ is accessible; open port `5672` |
| No `br-int` or `br-tun` bridges | Run: `sudo ovs-vsctl show` to check OVS setup |
| Instances can't get IP | Verify DHCP agent is running on controller |
| VXLAN tunnels not forming | Confirm `local_ip` matches compute node’s management IP |

---

## 📌 Summary Checklist

| Task | Status |
|------|--------|
| ☑️ Install `neutron-openvswitch-agent` | ✅ |
| ☑️ Edit `/etc/neutron/neutron.conf` | ✅ |
| ☑️ Choose correct networking option (1 or 2) | ✅ |
| ☑️ Configure `openvswitch_agent.ini` | ✅ |
| ☑️ Update `/etc/nova/nova.conf` | ✅ |
| ☑️ Restart `nova-compute` and `neutron-openvswitch-agent` | ✅ |
| ☑️ Verify agent status from controller | ✅ |

---

## 🚀 Next Steps

After Neutron is running on the compute node:

1. ➡️ [Create a Provider Network](https://docs.openstack.org/neutron/2025.1/admin/config-adv-routing.html)
2. ➡️ [Create a Self-Service Network (if using Option 2)](https://docs.openstack.org/neutron/2025.1/admin/config-routers.html)
3. ➡️ Launch an instance and test network connectivity

Example:
```bash
$ openstack server create --image cirros --flavor m1.tiny --network private-net --security-group default test-instance
```

---

🔗 **Official Docs**:  
[Neutron Compute Installation (Ubuntu)](https://docs.openstack.org/neutron/2025.1/install/compute-install-ubuntu.html)

🎯 You're now ready to run instances with full network isolation and security!

---



---
- Not yet Finish, Now Have a lot installation and configurations

To be Continue from bellow link:
https://docs.openstack.org/keystone/yoga/install/keystone-install-ubuntu.html
