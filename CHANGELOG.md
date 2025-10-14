# Changelog

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

