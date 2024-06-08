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
