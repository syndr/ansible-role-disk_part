# Changelog

## [Unreleased] - 2025-10-20
### Added
- **mount_path_type** parameter replacing mount_uuid_path with support for multiple path types:
  - `device`: Use raw device path (e.g., /dev/sdb1)
  - `uuid`: Use /dev/disk/by-uuid/ path (filesystem-based, changes on reformat)
  - `id`: Use /dev/disk/by-id/ path (hardware-based, survives reformatting)
  - `auto`: Try uuid first, fallback to id, then device (default, handles blank disks gracefully)
- Automatic by-id discovery task (get_device_id.yml) for hardware-based persistent paths
- Support for by-id paths with LVM volumes when specified
- Enhanced handling of blank unpartitioned disks that lack UUIDs
- Validation to ensure device path matches specified mount_path_type

### Changed
- **Default filesystem format changed from 'btrfs' to 'xfs'** for broader compatibility across enterprise Linux distributions
- **Default mount_path_type changed from 'uuid' to 'auto'** for better handling of blank disks

### Fixed
- LVM volume group name validation now properly detects mismatches between existing and requested VG names

### Removed
- mount_uuid_path parameter (replaced by mount_path_type)

## [Previous] - 2025-10-15
### Added
- mount_uuid_path option to mount non-LVM filesystems via /dev/disk/by-uuid for stability
- Automatic UUID discovery task (get_uuid.yml)
- Persisted fact file /etc/ansible/facts.d/disk_part.fact capturing disk config & UUIDs
- lvm_size_percent parameter
- Expanded Molecule resources: dynamic disk discovery, validation, new side effect & EC2 scenario (role-disk_part-ec2)
- Added requirements.txt with explicit dependencies (ansible-core >=2.20.0b2, molecule >=25.0, ara, etc.)

### Changed
- Default filesystem format: ext4 -> btrfs
- Default mount options: "" -> defaults
- Default state: present -> mounted
- Role references in Molecule updated from legacy name to disk_part; switched to syndr.molecule collection
- LVM mount entries now explicitly disable UUID mounting
- Improved validation & fact storage order (after UUID / LVM resolution)

### Fixed
- Correct handling when inspecting devices without children in LVM assertions
- Accurate device matching in verification for UUID vs LVM paths

### Breaking Changes
- Defaults changed (format=btrfs, state=mounted, mount_options=defaults, lvm now enabled by default) which may alter behavior of playbooks relying on previous implicit defaults.

## Improve validation and idempotence for disk management role

### 🎯 Fixed
- **Idempotence Issues**: Role now correctly handles already-configured disks
  - Added detection for matching filesystems and mount points
  - Prevents false failures when disks are already in desired state
  - Allows re-runs without requiring `force: true` for correct configs

- **LVM Multiple Device Support**: Fixed "Physical volume still in use" errors when multiple devices share the same volume group name
  - Enhanced `tasks/lvm.yml` with proper physical volume aggregation logic
  - Devices now correctly join existing volume groups instead of replacing existing physical volumes

### 🛡️ Added
- **Comprehensive Diskpart Validation**: Enhanced safety checks to prevent accidental data loss
  - Added filesystem detection and validation in `tasks/lvm.yml`
  - Added partition table and mount point validation 
  - Validation blocks dangerous operations with clear error messages and `force: true` guidance
  - Bidirectional protection: LVM creation over existing filesystems and filesystem creation over existing LVM

- **Production-Ready Testing Framework**: Complete Molecule test suite for AWS EC2
  - Added `molecule/role-disks-ec2/` scenario with comprehensive validation
  - Added `molecule/resources/side_effect.yml` with block/rescue error validation patterns
  - Enhanced `molecule/resources/verify.yml` with hostvar-driven disk configuration validation
  - Test coverage for both LVM and non-LVM scenarios with data integrity verification

### 🔧 Enhanced
- **Validation Logic in `tasks/check_state.yml`**:
  - Added `__disks_filesystem_matches` check for filesystem type matching
  - Added `__disks_device_mounted_correctly` for mount point verification
  - Added `__disks_is_already_configured` for non-LVM idempotence
  - Added `__disks_lvm_already_configured` for LVM idempotence
  - Enhanced `__disks_can_proceed` logic to handle idempotent operations
  - Improved error messages to differentiate between configuration mismatches
  - Fixed logic to prevent non-LVM operations on devices with "our" VG

- **LVM Configuration Management**: 
  - Fixed missing `lvm_size_percent` variable in `tasks/init.yml`
  - Fixed typo in LVM logical volume size configuration
  - Updated `defaults/main.yml` with improved LVM percentage sizing

- **Filesystem Validation Logic**: Enhanced null value handling in filesystem type detection

- **Test Infrastructure**:
  - Updated companion cube tagging from `molecule-notest` to `molecule-idempotence-notest`
  - Improved data integrity verification with better error handling and debug output
  - Enhanced side-effect test modularity with reusable task files

### 🧪 Updated
- **Side-Effect Test Error Messages**:
  - Updated all test assertions to match new validation message format
  - Tests now properly validate "Device contains" error messages
  - All dangerous operations blocked at `check_state.yml` level

### 📋 Technical Details
**Modified Files:**
- `tasks/check_state.yml` - Validation logic enhancements
- `tasks/lvm.yml` - Core LVM validation and aggregation logic
- `tasks/main.yml` - Role integration improvements
- `tasks/init.yml` - LVM size variable fixes
- `defaults/main.yml` - Updated LVM sizing configuration
- `meta/main.yml` - Role metadata enhancements

**New Test Infrastructure:**
- Complete Molecule test framework with AWS EC2 integration
- Comprehensive validation scenarios with proper error handling
- Data integrity verification and filesystem safety checks

