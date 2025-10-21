# Agent Integration Guide (AGENTS.md)

This document provides machine-oriented guidance for autonomous or semi-autonomous agents (LLMs, RPA systems, pipeline bots) interacting with this Ansible role repository.

## Repository Purpose
Manage Linux block devices with a single partition each, optionally configuring LVM, formatting filesystems, and mounting via systemd units or /etc/fstab. Persists resulting configuration & metadata as an Ansible fact file.

## High-Level Constraints
- Exactly one partition per physical block device supported (no multi-partition tables yet).
- LVM optional; when enabled, UUID mounting is automatically disabled.
- Idempotency expected; destructive operations only when `force: true`.

## Critical Paths & Files
- defaults/main.yml: Declares supported variables & defaults (authoritative schema for disks_partitions items).
- tasks/main.yml: Entry point; loops over partitions; writes fact file.
- tasks/manage_disk.yml (+ included task files): Core provisioning logic (LVM, filesystem, mounts, UUID discovery, cleanup).
- tasks/init.yml: Initializes internal fact aggregation lists.
- LICENSE: MIT (permissive; bots may reuse code snippets internally).
- CHANGELOG.md: Track evolutionary changes; agents should read for behavioral shifts (e.g., updated defaults).
- README.md: Human-facing usage examples (systemd vs fstab vs LVM).

## Role Invocation Pattern
Include via:
```yaml
- name: Configure disks
  vars:
    disks_partitions: [ ... ]
  ansible.builtin.include_role:
    name: disks  # Role name disk_part, but examples use 'disks'; verify runtime collection/namespace.
```
Effective defaults in `disks_partition_defaults` applied (agents merging user input MUST deep-merge with these values).

## Variable Schema (Operational Extraction)
Each item under `disks_partitions` supports:
- device (required)
- mount_path (required)
- format (default xfs) [btrfs|ext4|lvm|xfs|swap]
- format_options
- mount_type (systemd|fstab) default systemd
- mount_options (defaults)
- mount_path_type (device|uuid|id|auto, default auto)
- description
- force (bool)
- resizefs (bool) (not for swap)
- state (mounted|present|unmounted|remounted|absent|absent_from_fstab|ephemeral depending on mount_type)
- systemd_before / systemd_after
- LVM specific: lvm (bool), lvm_vg_name, lvm_lv_name, lvm_pv_options, lvm_vg_options, lvm_lv_options, lvm_pvresize (bool), lvm_size_percent (int 1-100)

Agents validating input should ensure:
- device path exists before provisioning (or is attachable in cloud pre-step).
- mount_path is absolute.
- format supported; if swap chosen, exclude resizefs.
- LVM or UUID-enabled mounts avoid device name conflicts due to kernel renaming.

## Generated Facts
File: /etc/ansible/facts.d/disk_part.fact (JSON list)
Contains merged per-disk configuration (post UUID/LVM resolution). Agents reading host state should parse this before proposing changes; prefer diff-based updates.

## Idempotency & Safety Guidelines for Agents
1. Test using the provided Molecule scenarios; replicate dynamic disk discovery if needed.
2. Only set `force: true` when intentional filesystem replacement is confirmed (irreversible format). Avoid for production devices unless migration plan documented.
3. When changing mount_path of existing device, ensure `disks_allow_mountpoint_changes: true` (default) else add remediation steps.
4. Avoid toggling between systemd and fstab mounts without a convergence plan (clean stale units or fstab entries).

## Common Automation Scenarios
- Add new data disk: append item with device, mount_path; rely on defaults for xfs & systemd.
- Convert existing ext4 fstab mount to systemd: set mount_type: systemd; ensure previous fstab entry removal (`state: absent_from_fstab`) if needed.
- Introduce LVM abstraction: set lvm: true, supply vg/lv names; format switched to filesystem (e.g., xfs) not 'lvm'.

## Detection Heuristics for Agents
- Presence of `lvm: true` -> treat device as PV; final mount path device will become /dev/vg-name/lv-name.
- If `mount_path_type: uuid` -> prefer /dev/disk/by-uuid reference (filesystem-based, changes on reformat)
- If `mount_path_type: id` -> prefer /dev/disk/by-id reference (hardware-based, survives reformatting)
- If `mount_path_type: auto` -> try uuid first, fallback to id, then device
- If `mount_path_type: device` -> use raw device path (e.g., /dev/sdb1)
- If fact records show differing device vs requested config device, migration may be incomplete.

## Change Impact Assessment (use CHANGELOG)
Defaults shifting (e.g., format ext4 -> btrfs -> xfs) can silently alter provisioning; agents upgrading role version should explicitly set prior defaults to maintain legacy behavior.

## Error Handling Patterns
Potential failure points:
- Missing device path -> filesystem task fails.
- Unsupported resizefs on swap -> guard with condition (agent must remove resizefs flag).
- Mount conflicts -> role attempts unmount when force: true; agents should review planned overlapping mounts before enabling force.

## Safe Rollback Strategy for Agents
- Molecule scenarios test in isolation; this test environment can be used to validate rollback.
- Use `molecule destroy -s <scenario name>` to clean up test VMs.

## Testing & Validation Recommendations
- Use Molecule scenarios (directory present) as blueprint; replicate dynamic disk discovery.
- Pre-flight: ansible.builtin.assert ensures disks_partitions is list; agents should not override type.

## Interaction Etiquette for AI Agents
- Minimize diff footprint: modify only necessary lines in YAML variable files/playbooks.
- Preserve comments & folding markers in defaults/main.yml.
- Do not alter LICENSE, CHANGELOG entries retroactively; append new entries under [Unreleased].
- Use to_nice_json formatting when scripting fact rewrites for consistency.

## Extension Opportunities (For Future Agents)
- Multi-partition support (extend schema with partition_number, size, GPT layout directives).
- Encryption layer (dm-crypt/LUKS parameters pre-filesystem).

## Quick Reference (Machine-Readable JSON Schema Draft)
```json
{
  "disks_partitions.item": {
    "device": "string:path",
    "mount_path": "string:absolute path",
    "format": {"enum": ["btrfs", "ext4", "lvm", "xfs", "swap"], "default": "xfs"},
    "format_options": "string",
    "mount_type": {"enum": ["systemd", "fstab"], "default": "systemd"},
    "mount_options": {"type": "string", "default": "defaults"},
    "mount_path_type": {"enum": ["device", "uuid", "id", "auto"], "default": "auto"},
    "description": "string",
    "force": {"type": "boolean", "default": false},
    "resizefs": {"type": "boolean", "default": false},
    "state": "string",
    "systemd_before": "string",
    "systemd_after": "string",
    "lvm": {"type": "boolean", "default": true},
    "lvm_vg_name": "string",
    "lvm_lv_name": "string",
    "lvm_pv_options": "string",
    "lvm_vg_options": "string",
    "lvm_lv_options": "string",
    "lvm_pvresize": {"type": "boolean", "default": true},
    "lvm_size_percent": {"type": "integer", "minimum": 1, "maximum": 100, "default": 100}
  }
}
```

## Attribution
Author: [syndr](https://github.com/syndr). License: MIT.

---
Generated: 2025-10-16T21:24:39Z
