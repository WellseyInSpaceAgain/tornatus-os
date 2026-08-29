# Milestone 0 Fedora 44 VM log

This file records the manual Milestone 0 experiment. It is an observation log,
not an automated installation recipe.

## VM

```text
Name:       tornatus-m0-f44
Firmware:   UEFI
CPU:        4 vCPU
Memory:     8 GiB
Disk:       32 GiB virtio
Installer:  Fedora Server 44 DVD
Hostname:   tornatus-m0
User:       tornatus (administrator)
```

The Fedora installer created a 600 MiB EFI partition, a 2 GiB XFS `/boot`, and
a Btrfs root subvolume named `root` on `/dev/vda3`.

A powered-off hypervisor checkpoint named `fresh-f44-before-snapper` was made
before Snapper configuration.

## Packages installed

```bash
sudo dnf install snapper tukit dracut-transactional-update
```

Fedora installed tukit 6.0.0, Snapper 0.13.0, and their dependencies. The
Snapper `50-etc` plugin was included in the `tukit` package.

## Observed snapshot sequence

| Snapper number | Purpose | Outcome |
|---:|---|---|
| 0 | Virtual current system | Listable, but not usable as a tukit base |
| 1 | Manually created initial base | First real read-only snapshot |
| 2 | First successful no-op tukit transaction | First booted transactional root |
| 3 | Second no-op transaction | Used as rollback target |
| 4 | Added `/etc/tornatus-m0-transaction` | Change activated, then rolled back |
| 5 | Automatic timeline snapshot | Created by Snapper's enabled timer |
| 6 | Removed `rootflags=subvol=root` from `/etc/kernel/cmdline` | Known-good normal boot |
| 7 | Added an attempted `/.snapshots` fstab entry | Entered emergency mode on boot |

Numeric Btrfs IDs observed during the experiment included:

```text
256  original Fedora root
258  root/.snapshots repository
259  Snapper snapshot 1
261  Snapper snapshot 2
262  Snapper snapshot 3
263  Snapper snapshot 4
265  Snapper snapshot 6
```

## Successful Milestone 0 proof

The experiment demonstrated:

1. a change created in a new tukit snapshot was absent from the running root;
2. the change existed in the pending snapshot;
3. reboot activated that snapshot and made the change visible;
4. `tukit rollback 3` selected the earlier root; and
5. reboot activated snapshot 3 and removed the change from view.

See `docs/architecture.md` for the detailed findings and caveats.

## Current VM state and recovery

The latest boot attempted snapshot 7 and entered emergency mode after adding
this entry to its `/etc/fstab`:

```text
UUID=806b48e0-565d-4369-9649-dc3ec39c8169 /.snapshots btrfs subvol=root/.snapshots,compress=zstd:1 0 0
```

The equivalent interactive mount had succeeded, so the boot failure remains
unexplained. The root account is locked, preventing login at the emergency
prompt.

Snapshot 6 is the last known-good root. A non-destructive recovery path is:

1. reboot to GRUB;
2. edit the normal Fedora entry for one boot;
3. add this argument to the kernel line:

   ```text
   rootflags=subvol=root/.snapshots/6/snapshot
   ```

4. boot snapshot 6;
5. temporarily mount the shared repository if necessary:

   ```bash
   sudo mount -t btrfs \
     -o subvol=root/.snapshots,compress=zstd:1 \
     UUID=806b48e0-565d-4369-9649-dc3ec39c8169 \
     /.snapshots
   ```

6. inspect logs from the failed snapshot 7 boot before changing or deleting
   anything;
7. select snapshot 6 as default after the evidence has been collected.

The powered-off `fresh-f44-before-snapper` checkpoint remains the broader
fallback.
