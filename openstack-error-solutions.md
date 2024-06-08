#
## OpenStack Error, Solutions and Troubleshooting
#
####
<b> 1. Error:</b> Something went wrong! An unexpected error has occurred. Try refreshing the page. If that doesn't help, contact your local administrator.
####
    service apache2 restart
    service memcached restart
####
<b> 2. Error:</b> If can't reach host to instance
####
    nano /etc/neutron/dhcp_agent.ini
####
Search
####
    isolated
####
    enable_isolated_metadata = true
####
    systemctl restart neutron-dhcp-agent
####

<b> 3. Error:</b> When luanch a instance then show the error timeout
####
Show nova-compute.log
####
    tail -f /var/log/nova/nova-compute.log
####
    nano /etc/nova/nova.conf.d/100-nova.conf
####
    block_device_allocate_retries = 300
    block_device_allocate_retries_interval = 3
####
<b> 4. Error:</b> OpenStack-Exceeded-Maximum-Number-Of-Retries-Exhausted-All-Hosts-Available-For-Retrying-Build-Failures-For-Instance.
####
<i>Refference:</i> https://www.informaticar.net/openstack-exceeded-maximum-number-of-retries-exhausted-all-hosts-available-for-retrying-build-failures-for-instance/
####
    tail -f /var/log/cinder/volume.log
####
    tail -f /var/log/nova/nova-conductor.log
####
<b> 5. Error:</b> NoValidHost: No valid host was found. There are not enough hosts available
####
Solution:<br>
https://stackoverflow.com/questions/45511739/novalidhost-no-valid-host-was-found-there-are-not-enough-hosts-available
####
Edit /etc/nova/nova.conf in all the compute node and modify as per your application requirement .
####
    nano /etc/nova/nova.conf
####
Double of Physical Core can be used for total Instance <br>
Double of Total Memory can be used for total Instance
####
    cpu_allocation_ratio = 2.0
    ram_allocation_ratio = 2.0     
####
Restart Nova and Nova-Scheduler in all the Compute Node
####
    systemctl restart openstack-nova-*
    systemctl restart openstack-nova-scheduler.service

