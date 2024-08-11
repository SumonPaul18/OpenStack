### Configure an NFS storage back end
Reference: 
- https://docs.openstack.org/cinder/rocky/admin/blockstorage-nfs-backend.html
- https://kimizhang.wordpress.com/2015/02/12/nfs-backend-for-openstack-glancecinderinstance-store/
- https://www.server-world.info/en/note?os=CentOS_7&p=openstack_train2&f=10
- https://access.redhat.com/articles/1323213
#
### Managing OpenStack, Using OpenStack Cli
Reference: 
- https://docs.redhat.com/en/documentation/red_hat_openstack_platform/17.0/html/creating_and_managing_instances/assembly_providing-public-access-to-an-instance_instances#proc_creating-an-instance-with-ssh-access_instances
#
### How to install OpenStackClient for Linux
Reference:
- https://destine-data-lake-docs.data.destination-earth.eu/en/latest/openstackcli/How-to-install-OpenStackClient-for-Linux.html
### OpenStack Shared File Systems
https://docs.openstack.org/project-install-guide/shared-file-systems/newton/install-controller-rdo.html
### Magnum Installation Guide
https://docs.openstack.org/magnum/2024.1/install/install-rdo.html
### Install OpenStack client on AlmaLinux

####
Update Systems and installing Python and PIP
####
    dnf update
    dnf install python3.8 python3.8-pip -y
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
####
Add the following lines:
####
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
####
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
####
    #pip3 uninstall python-openstackclient

    
#
## Manage OpenStack Services
####
    openstack service list
####
Restart Nova services.
####
    systemctl restart openstack-nova-compute

    systemctl restart openstack-nova-api

    systemctl restart openstack-nova-scheduler
####
    . keystonerc_admin
####
    openstack-status
####
    openstack-service status
####
    openstack-service start
####
    openstack-service restart
####
    openstack-service stop
####
    systemctl status rabbitmq-server.service
####
    systemctl start rabbitmq-server
####
    systemctl restart openstack-nova-network
####
    systemctl enable rabbitmq-server.service
####
    systemctl restart rabbitmq-server.service
####
    openstack-service status cinder
####
List the Compute services:
####
    openstack compute service list
####
Disable a nova service:
####
    openstack compute service set --disable --disable-reason "trial log" compute nova-compute
####
Enable the service:
####
    openstack compute service set --enable compute nova-compute

####
    systemctl start neutron-metadata-agent.service
####
    systemctl enable neutron-metadata-agent.service
####
    openstack-service status glance | column -t | grep active
####
    openstack-service restart glance    -restart glance all 
####
    systemctl status openstack-glance-api.service
####
    service rabbitmq-server stop
####
    service rabbitmq-server start
####
    service openstack-nova-compute restart
####
    #openstack-service list |xargs openstack-service disable
####
    #openstack-service list |xargs openstack-service enable
####
    #openstack-service list |xargs openstack-service stop
####
    #openstack-service list |xargs openstack-service
####
    #restart nova and nova-scheduler in all the compute node
####
    systemctl restart openstack-nova-*
####
    systemctl restart openstack-nova-scheduler.service
####
    systemctl restart httpd
    systemctl restart memcached
####
    systemctl status openvswitch
####
    systemctl restart openvswitch
####
    systemctl start openvswitch libvirtd neutron-openvswitch-agent openstack-nova-compute 
####
    systemctl restart openvswitch libvirtd neutron-openvswitch-agent openstack-nova-compute 
####
    systemctl enable openvswitch libvirtd neutron-openvswitch-agent openstack-nova-compute 
####
    systemctl restart openvswitch libvirtd neutron-openvswitch-agent openstack-nova-compute
####
    systemctl start openstack-nova-compute openstack-neutron openstack-neutron-openvswitch
####
    systemctl restart openstack-nova-compute openstack-neutron openstack-neutron-openvswitch
####
    systemctl restart openstack* neutron* libvirtd
####
    systemctl status openstack* neutron* libvirtd
####
    systemctl restart neutron*
####
    systemctl restart openstack-nova-compute.service
####
#
#### Reboot an Instance
Soft Reboot an Instance
####
    openstack server reboot SERVER/ID
####
Hard Reboot an Instance
####
    openstack server reboot --hard SERVER/ID
################
###    Integrade OpenStack Cinder with NFS Storage
####
    dnf -y install nfs-utils
####
    cat <<EOF | sudo tee /etc/idmapd.conf
    Domain = paulco.xyz
    EOF
####
    systemctl status rpcbind
    systemctl start rpcbind
    systemctl enable rpcbind
####
    cat <<EOF | sudo tee -a /etc/cinder/nfs_shares
    192.168.0.96:/nfs-share
    EOF
####
    chown root:cinder /etc/cinder/nfs_shares
    #chmod 0640 /etc/cinder/nfs_shares
    chmod 777 /etc/cinder/nfs_shares
    chgrp cinder /etc/cinder/nfs_shares
    systemctl restart openstack-cinder-volume
    #chown -R cinder. /var/lib/cinder/mnt
####
    cat <<EOF | sudo tee -a /etc/cinder/cinder.conf
    # add follows in [DEFAULT] section
    enabled_backends = nfs
    # add follows to the end
    [nfs]
    volume_driver = cinder.volume.drivers.nfs.NfsDriver
    nfs_shares_config = /etc/cinder/nfs_shares
    #nfs_mount_point_base = $state_path/mnt
    EOF

####
    service openstack-cinder-api restart
    service openstack-cinder-backup restart
    service openstack-cinder-scheduler restart
    service openstack-cinder-volume restart
####
    openstack-service status | grep -i cinder
    service openstack-cinder-api status
    service openstack-cinder-backup status
    service openstack-cinder-scheduler status
    service openstack-cinder-volume status
####
Deatch Volume of OpenStack Instance
####
    openstack volume list
####
Example: openstack volume set --state available --detached cdd25f1a-f09c-43fb-bde4-a3c6e80d33b8
####
    openstack volume set --state available --detached volume_id/volume_name
####
    openstack availability zone list
####
    openstack volume delete my-new-volume
####
Run a shell script at startup
####
    dnf install crontabs -y
####
    systemctl status crond.service
    systemctl start crond.service
    systemctl enable crond.service
####
    chmod +x /home/user/startup.sh
####
Setup Jobs a crontab for it:
####
List of Crontab Jobs
    crontab -l
####
Adding/Edit jobs in Crontab
####
    crontab -e
    @reboot  /home/user/startup.sh
####
Adding a jobs in Crontab using CLI
####
    crontab -e '*/5 * * * * /path/to/your/script.sh'
####
    crontab -l | grep -v '^$' ; echo '@reboot /root/build-openstack.sh' | crontab -
####
Deleting an existing job/task in Crontab
Delete all jobs/tasks
####
    crontab -r
####
Remove the specific entry
crontab -l | grep -v '<SPECIFICS OF YOUR SCRIPT HERE>' | crontab -
####
    crontab -l | grep -v '* * * * * /usr/local/bin/download_weather.py' | crontab -
####
Install and Use Screen in Linux
####
    dnf install screen -y
    screen –version
    screen -S Openstack
    screen -ls
    screen -r OpenStack/ID  

#
### Configure OpenStack Instances Autostart after reboot
#
Open Nova configuration file
####
    nano /etc/nova/nova.conf
####
Add below line under the [DEFAULT] section: <br>
By Default we find <b>#resume_guests_state_on_host_boot = false </b>  (So, we need to make it Uncomment and True).
####
    resume_guests_state_on_host_boot = true
####
    sudo systemctl restart openstack-nova-compute
#






