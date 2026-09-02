# Tornatus

Tornatus is an experimental Fedora-based, image-oriented Linux distribution
exploring openSUSE-inspired transactional system management using Btrfs,
Snapper and tukit.

## Milestone 0 status

Milestone 0 has been demonstrated on a Fedora 44 VM. The experiment proved
that Tornatus can:

- stage a system change in a new read-only root snapshot without modifying the
  running root;
- activate the new snapshot with a normal reboot;
- roll back to an earlier snapshot and activate it on reboot; and
- repeat the transaction and boot cycle without manual GRUB edits or mounts.

The Fedora boot configuration must allow the Btrfs default subvolume to select
the root instead of pinning `rootflags=subvol=root`. The shared Snapper
repository must also be mounted at `/.snapshots` from every transactional root.
Both requirements were validated through snapshot 10, which is the VM's active
and default snapshot.

Persistent journald storage on read-only transactional roots remains an open
integration issue: volatile logs from a failed boot were lost after reboot.

See [the architecture notes](docs/architecture.md) for the findings and
[the VM experiment log](vm/README.md) for the observed snapshot sequence and
recovery procedure.
