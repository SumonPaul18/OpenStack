
# Install OpenStack Manually 
---
Reference: 
1) https://www.youtube.com/playlist?list=PLVV1alynPj3E4s4Nt2VM2J5Q0gl69WDR7

+++++++++++++
#Requirement:
++++++++++++++
#Two VMs 
1. Controller
2. Compute

#Operating System
Ubuntu Latest 

#Change Hostname

#Configure IP

vi /etc/netplan/50

auto enp0s3
iface enp0s3 inet static
address 10.0.2.40
netmask 255.255.255.0
gateway 10.0.2.2

auto enp0s8
iface enp0s8 inet dhcp

service networking restart

#update hosts file

10.0.2.40 controller
10.0.2.50 compute

#Configure Password less authentication on both vms
ssh-keygen

#go to compute node
#change root password
passwd root

vi /etc/ssh/sshd_config
#uncomment
PermitRootLogin yes

service sshd restart

#copy ssh-key from controller to compute node

ssh-copy-id -i root@compute

#Install NTP

apt install chrony -y

vi /etc/chrony/chrony.conf

#comment default ntp server
#server 0.asia.pool.ntp.org
#server 1.asia.pool.ntp.org
#server 2.asia.pool.ntp.org
#server 3.asia.pool.ntp.org

#add own ntp server its host ip
server 192.168.0.89 iburst
allow 192.168.106.15/24

service chrony restart

#go to compute node ntp configure
vi /etc/chrony/chrony.conf

#comment default ntp server
#pool 0.ubuntu.pool.ntp.org
#pool 1
#pool 2

#add controller server ip as ntp server
server controller iburst

service chrony restart

#verify ntp server
chronyc sources

#Now install OpenStack
> openstack.org
> Docs
> Installation Guide
> choose version and
Following [Environment] Section step by step:

## 1st Create Environment for Install OpenStack ##

#add OpenStack Packages on both node
```
add-apt-repository cloud-archive:epoxy
```
#install openstackclient on both node
```
apt install python3-openstackclient
```
#Install and configure SQL Database
```
apt install mariadb-server python3-pymysql -y
```
```
vi /etc/mysql/mariadb.conf.d/99-openstack.cnf
```
[mysqld] 
```
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
#### choose a suitable password [****] for the database root account

```
mysql_secure_installation
```

#Install and configure rabbitmq-server
```
apt install rabbitmq-server -y
```
```
service rabbitmq-server restart
```
#rabbitmqctl add_user openstack ubuntu
```
rabbitmqctl add_user openstack openstack
```
#Permit configuration, write, and read access for the openstack user
```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
#Install and configure memcached
```
apt install memcached python3-memcache -y
```
#edit /etc/memcached.conf file
```
vi /etc/memcached.conf
```
```
-l 192.168.106.15
```
```
service memcached restart
```
##No need to install Etcd

## Now Following [ Install OpenStack services ] Option Step by Step ##

> go to Minimal deployment for your version as my zed

#Install and configure keystone

#Create Database for keystone

#access mysql
```
mysql
```
#login mysql console show: MariaDB [(none)]> 
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
#exit from sql

#Install keystone
```
apt install keystone -y
```
#Edit the /etc/keystone/keystone.conf
```
vi /etc/keystone/keystone.conf
```
#add new connection under [database] section and comment default connection.

[database]
```
connection = mysql+pymysql://keystone:ubuntu@controller/keystone
```
#[token] section, configure the Fernet token provider.

[token]
```
provider = fernet
```
#Populate the Identity service database.
```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
#Initialize Fernet key repositories.
```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
#Bootstrap the Identity service.
```
keystone-manage bootstrap --bootstrap-password openstack \
  --bootstrap-admin-url http://pmta:5000/v3/ \
  --bootstrap-internal-url http://pmta:5000/v3/ \
  --bootstrap-public-url http://pmta:5000/v3/ \
  --bootstrap-region-id RegionOne
 ``` 
  
#Configure the Apache HTTP server

#Edit the /etc/apache2/apache2.conf
```
vi /etc/apache2/apache2.conf
```
#add your ServerName under Global configuration section.
```
ServerName controller
```
```
service apache2 restart
```
# Configure the administrative account by setting the proper environmental variables:

>>> Not yet Finish, Now Have a lot installation and configurations >>>

To be Continue from bellow link:
https://docs.openstack.org/keystone/yoga/install/keystone-install-ubuntu.html
