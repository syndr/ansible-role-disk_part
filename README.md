disks
=========

Configure disks on the target host. This includes:
- Partitioning and formatting
- Mount points

> [!IMPORTANT]  
> This role only supports 1 partition per block device at this time!

Requirements
------------

Collections:
- community.general
- ansible.posix

The desired disk must already be attached to the target machine.

Role Variables
--------------

```yaml.ansible
######## Disk Partitions #######################################################
#
# The disks_partitions variable contains a list of dictionaries which define
# individual partitions that are to be managed.
#
# Each listed partition supports the following options:
#   GENERAL
#   - description (optional): The description of the mount, if systemd type is used (string)
#   - device (required): The linux system device to be managed
#   - force (optional): Allow overwriting of existing filesystems 💀
#   - format (optional): Format for the partition (btrfs/ext4/lvm/xfs/swap)
#   - format_options (optional): Options to be passed to the mkfs command (string)
#   - mount_path (required): The file system path at which to mount the partition
#   - mount_type (optional): Method to use to mount the partition (fstab/systemd)
#   - mount_options (optional): Options to be used when mounting the filesystem (string)
#   - mount_path_type (optional): Type of persistent device path to use (device/uuid/id/auto, default: auto)
#      * device: Use raw device path (e.g., /dev/sdb1)
#      * uuid: Use /dev/disk/by-uuid/ path (filesystem-based, changes on reformat)
#      * id: Use /dev/disk/by-id/ path (hardware-based, survives reformatting)
#      * auto: Try uuid first, fallback to id, then device (useful for unformatted disks)
#   - resizefs (optional): Grow the filesystem to match the size of the block device (true/false)
#      * not supported for swap format
#   - state (optional): Existence of the partition
#     - `mount_type: fstab`: (absent/absent_from_fstab/mounted/present/unmounted/remounted/ephemeral)
#     - `mount_type: systemd`: (absent/present/mounted/unmounted)
#
#   SYSTEMD
#   - systemd_before (optional): Systemd service which this filesystem should be mounted before (string)
#   - systemd_after (optional): Systemd service which this filesystem should be mounted after (string)
#
#   LVM
#   - lvm (optional): Should a LVM partitioning scheme be used (true/false)
#   - lvm_vg_name (optional): The name of the LVM volume group (string)
#   - lvm_lv_name (optional): The name of the LVM logical volume (string)
#   - lvm_pv_options (optional): Options to be passed to the pvcreate command (string)
#   - lvm_vg_options (optional): Options to be passed to the vgcreate command (string)
#   - lvm_lv_options (optional): Options to be passed to the lvcreate command (string)
#   - lvm_pvresize: (optional): Should LVM volumes be expanded to match block device size (true/false)
#

disks_partitions:
  - device: /dev/sdx
    format: ext4
    mount_path: /mnt/example_disk
    mount_type: systemd
    systemd_before: docker.service
    description: The bestest disk evarr!
    force: false


# Default values ↓
disks_partition_defaults:
  format: btrfs
  format_options: ""
  mount_type: systemd
  mount_options: defaults
  mount_path_type: auto
  force: false
  resizefs: false
  state: mounted
  systemd_before: ""
  systemd_after: ""
  description: "Disk managed by Ansible"
  lvm: true
  lvm_vg_name: data
  lvm_lv_name: data
  lvm_vg_options: ""
  lvm_pv_options: ""
  lvm_lv_options: ""
  lvm_pvresize: true
  lvm_size_percent: 100
```

> [!IMPORTANT]
> Setting `state: absent` will remove the mount for a disk only! Partition table and attachement status will not be modified.

Dependencies
------------

None

Example Playbook
----------------

Example configuration using a traditional `/etc/fstab` mount:  
```yaml.ansible
- name: Make the disks
  hosts: all
  tasks:
    - name: Configure disks
      ansible.builtin.include_role:
        name: disks
      vars:
        disks_partitions:
          - device: /dev/sdf
            format: ext4
            mount_path: /mnt/test
            mount_type: fstab
            mount_options: defaults,noatime
            mount_path_type: auto  # Mounts using /dev/disk/by-uuid/{uuid}
```

Example configuration using a systemd mount unit:  
```yaml.ansible
- name: Make the disks
  hosts: all
  tasks:
    - name: Configure disks
      ansible.builtin.include_role:
        name: disks
      vars:
        disks_partitions:
          - device: /dev/sdf
            format: ext4
            mount_path: /mnt/test
            mount_type: systemd
```

Example configuration with LVM:
```yaml.ansible
- name: Make the disks with LVM
  hosts: all
  tasks:
    - name: Configure disks
      ansible.builtin.include_role:
        name: disks
      vars:
        disks_partitions:
          - device: /dev/sdf
            format: xfs
            mount_path: /mnt/data
            mount_type: systemd
            lvm: true
            lvm_vg_name: data
            lvm_lv_name: storage
            # Can still use mount_path_type: uuid or id with LVM
```

Example using hardware-based by-id paths (useful for unformatted disks):
```yaml.ansible
- name: Configure disk with by-id path
  hosts: all
  tasks:
    - name: Configure disks
      ansible.builtin.include_role:
        name: disks
      vars:
        disks_partitions:
          - device: /dev/sdb
            format: btrfs
            mount_path: /mnt/stable
            mount_type: systemd
            mount_path_type: id  # Uses /dev/disk/by-id/{hardware-id}
            # Survives reformatting, unlike UUID
```

Example using auto mode for blank disks:
```yaml.ansible
- name: Configure potentially blank disk
  hosts: all
  tasks:
    - name: Configure disks
      ansible.builtin.include_role:
        name: disks
      vars:
        disks_partitions:
          - device: /dev/sdc
            format: ext4
            mount_path: /mnt/newdisk
            mount_type: systemd
            mount_path_type: auto  # Tries uuid, falls back to id, then device
            # Useful for disks that may not have a filesystem yet
```

Ansible Facts
-------------

This role saves disk configuration to `/etc/ansible/facts.d/disk_part.fact` on the target host. The facts include:

- All configuration parameters for each disk
- UUID and device ID values for disks (when `mount_path_type` is set)
- Actual device paths used for mounting

Example fact data:
```json
[
  {
    "device": "/dev/disk/by-uuid/5f2c38d2-c5d5-47cd-a2e9-2f023294b4d0",
    "uuid": "5f2c38d2-c5d5-47cd-a2e9-2f023294b4d0",
    "format": "ext4",
    "mount_path": "/mnt/test",
    "mount_type": "fstab",
    "mount_path_type": "uuid",
    "lvm": false
  },
  {
    "device": "/dev/vg-data/lv-storage",
    "format": "xfs",
    "mount_path": "/mnt/data",
    "mount_type": "systemd",
    "mount_path_type": "device",
    "lvm": true,
    "lvm_vg_name": "data",
    "lvm_lv_name": "storage"
  }
]
```

License
-------

MIT

Author Information
------------------

- [@syndr](https://github.com/syndr/)

