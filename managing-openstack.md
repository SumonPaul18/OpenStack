### Configure an NFS storage back end
Reference: 
####
    > https://docs.openstack.org/cinder/rocky/admin/blockstorage-nfs-backend.html
    > https://kimizhang.wordpress.com/2015/02/12/nfs-backend-for-openstack-glancecinderinstance-store/
    > https://www.server-world.info/en/note?os=CentOS_7&p=openstack_train2&f=10
    > https://access.redhat.com/articles/1323213

#

### Install OpenStack client on AlmaLinux

####

Installing Python and PIP
    dnf update
    dnf install python38 python38-pip -y
    pip3 install --upgrade pip -y

Checking PIP version
####
    pip3 --version

####
Installing OpenStack client
    pip3 install python-openstackclient

####
    pip3 show python-openstackclient
Checking the OpenStack client version
####
    source ~/.bashrc
    openstack --version
Configuring the OpenStack client
####
    nano ~/.keystonerc
Add the following lines:
    unset OS_SERVICE_TOKEN
    export OS_USERNAME='admin'
    export OS_PASSWORD='adminpassword'
    export OS_AUTH_URL=http://localhost:5000/v3
    export PS1='[\u@\h \W(keystone_admin)]\$ '

    export OS_PROJECT_NAME=admin
    export OS_USER_DOMAIN_NAME=Default
    export OS_PROJECT_DOMAIN_NAME=Default
    export OS_IDENTITY_API_VERSION=3

####
    source ~/.keystonerc

Verifying the OpenStack client installation
####
    openstack server list
    openstack network list
    openstack image list

####
update the OpenStack client
####
    pip3 install --upgrade python-openstackclient

####
How to uninstall the OpenStack client
    #pip3 uninstall python-openstackclient

    
################################

openstack service list

#OpenStack Cinder Storage (NFS)
dnf -y install nfs-utils
cat <<EOF | sudo tee /etc/idmapd.conf
Domain = paulco.xyz
EOF

systemctl status rpcbind
systemctl start rpcbind
systemctl enable rpcbind

cat <<EOF | sudo tee /etc/cinder/nfs_shares
192.168.0.96:/nfs-share
EOF

chown root:cinder /etc/cinder/nfs_shares
#chmod 0640 /etc/cinder/nfs_shares
chmod 777 /etc/cinder/nfs_shares
chgrp cinder /etc/cinder/nfs_shares
systemctl restart openstack-cinder-volume
#chown -R cinder. /var/lib/cinder/mnt

cat <<EOF | sudo tee /etc/cinder/cinder.conf
# add follows in [DEFAULT] section
enabled_backends = nfs
# add follows to the end
[nfs]
volume_driver = cinder.volume.drivers.nfs.NfsDriver
nfs_shares_config = /etc/cinder/nfs_shares
#nfs_mount_point_base = $state_path/mnt
EOF


service openstack-cinder-api restart
service openstack-cinder-backup restart
service openstack-cinder-scheduler restart
service openstack-cinder-volume restart

openstack-service status | grep -i cinder
openstack volume service list

service openstack-cinder-api status
service openstack-cinder-backup status
service openstack-cinder-scheduler status
service openstack-cinder-volume status

#Deatch Volume in OpenStack
openstack volume list
#openstack volume set --state available --detached cdd25f1a-f09c-43fb-bde4-a3c6e80d33b8
openstack volume set --state available --detached volume_id/volume_name

openstack availability zone list
openstack volume delete my-new-volume



#run a shell script at startup

#dnf install crontabs -y
#systemctl status crond.service
#systemctl start crond.service
#systemctl enable crond.service

chmod +x /home/user/startup.sh

#set a crontab for it:
crontab -l
crontab -e
@reboot  /home/user/startup.sh

#crontab -e '*/5 * * * * /path/to/your/script.sh'

crontab -l | grep -v '^$' ; echo '@reboot /root/build-openstack.sh' | crontab -



#Deleting an existing job/task in Crontab

#delete all jobs/tasks
#crontab -r

#remove the specific entry
crontab -l | grep -v '<SPECIFICS OF YOUR SCRIPT HERE>' | crontab -
crontab -l | grep -v '* * * * * /usr/local/bin/download_weather.py' | crontab -

#Install and Use Screen in Linux

dnf install screen -y
screen –version
screen -S Openstack
screen -ls
screen -r 2898.upgrade  











