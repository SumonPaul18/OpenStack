# Managing OpenStack Storage, Instance Volumes, and Backups. Let's go through your questions one by one.

### 1. Where are instance volumes stored?

When you create an instance in OpenStack without selecting `create new volume,` the instance uses ephemeral storage. This storage is temporary and is deleted when the instance is terminated. The ephemeral storage is typically located in the `/var/lib/nova/instances` directory on the compute node.

For persistent storage, OpenStack uses the Cinder service to manage volumes. These volumes are stored on block storage devices and can be attached to instances. By default, Cinder uses LVM (Logical Volume Manager) as the backend, but you can configure it to use other storage backends like Ceph, NFS, or third-party storage solutions.

### 2. How to automatically backup instance volumes?

To automatically backup instance volumes in OpenStack, you can use the Cinder service's backup feature. Here are the steps to create a backup:

1. **Create a Backup**:
   ```bash
   openstack volume backup create --name <backup_name> <volume_id>
   ```
   You can also create incremental backups by adding the `--incremental` flag.

2. **Restore a Backup**:
   ```bash
   openstack volume backup restore <backup_id> <volume_id>
   ```

For automation, you can set up a cron job or use OpenStack's built-in scheduling services to run these commands at regular intervals.

### 3. How to use external storage with OpenStack for storing instance volumes?

You can integrate external storage solutions with OpenStack using the Cinder service. Here are the general steps:

1. **Configure Cinder to Use External Storage**:
   - Edit the `/etc/cinder/cinder.conf` file to add the configuration for your external storage backend. For example, to use NFS:
     ```ini
     [DEFAULT]
     enabled_backends = nfs

     [nfs]
     volume_driver = cinder.volume.drivers.nfs.NfsDriver
     nfs_shares_config = /etc/cinder/nfs_shares
     ```
   - Create the `/etc/cinder/nfs_shares` file and add the NFS server details:
     ```plaintext
     nfs-server:/exported/path
     ```

2. **Restart Cinder Services**:
   ```bash
   systemctl restart openstack-cinder-api
   systemctl restart openstack-cinder-scheduler
   systemctl restart openstack-cinder-volume
   ```

3. **Create Volumes Using the External Storage**:
   When creating a volume, specify the backend:
   ```bash
   openstack volume create --size <size> --volume-type nfs <volume_name>
   ```

### Summary

- **Instance Volumes**: Ephemeral storage is used by default, located at `/var/lib/nova/instances`. Persistent volumes are managed by Cinder.
- **Automatic Backup**: Use Cinder's backup feature and automate with cron jobs or scheduling services.
- **External Storage**: Configure Cinder to use external storage backends like NFS, Ceph, etc.

Reference:
(1) Safeguarding Your OpenStack Instance: Complete Guide to ... - Superuser. https://superuser.openinfra.dev/articles/safeguarding-your-openstack-instance-complete-guide-to-manually-backing-up-ephemeral-and-block-storage/.

(2) Manage volumes - OpenStack. https://docs.openstack.org/cinder/rocky/cli/cli-manage-volumes.html.

(3) Chapter 2. Block Storage and Volumes - Red Hat. https://docs.redhat.com/en/documentation/red_hat_openstack_platform/11/html/storage_guide/ch-cinder.

(4) Back up and restore volumes and snapshots — cinder 25.1.0 ... - OpenStack. https://docs.openstack.org/cinder/latest/admin/volume-backups.html.

(5) Back up and restore volumes and snapshots - OpenStack. https://docs.openstack.org/cinder/wallaby/admin/blockstorage-volume-backups.html.
