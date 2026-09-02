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
| 7 | Added mistyped `./snapshots` fstab target | Entered emergency mode; preserved for inspection |
| 8 | Automatic timeline snapshot | Created by Snapper's enabled timer |
| 9 | Corrected the target to `/.snapshots` | Normal boot; repository mounted automatically |
| 10 | No-op transaction based on snapshot 9 | Normal boot; verified repeatable steady state |

Numeric Btrfs IDs observed during the experiment included:

```text
256  original Fedora root
258  root/.snapshots repository
259  Snapper snapshot 1
261  Snapper snapshot 2
262  Snapper snapshot 3
263  Snapper snapshot 4
265  Snapper snapshot 6
268  Snapper snapshot 9
269  Snapper snapshot 10
```

## Successful Milestone 0 proof

The experiment demonstrated:

1. a change created in a new tukit snapshot was absent from the running root;
2. the change existed in the pending snapshot;
3. reboot activated that snapshot and made the change visible;
4. `tukit rollback 3` selected the earlier root; and
5. reboot activated snapshot 3 and removed the change from view.

The experiment subsequently demonstrated a second complete transaction and
normal boot with no GRUB edit or manual repository mount. See
`docs/architecture.md` for the detailed findings and caveats.

## Snapshot 7 failure and recovery

Snapshot 7 entered emergency mode because its `/etc/fstab` target was mistyped:

```text
UUID=806b48e0-565d-4369-9649-dc3ec39c8169 ./snapshots btrfs subvol=root/.snapshots,compress=zstd:1 0 0
```

The relative `./snapshots` target was not the intended absolute hidden path
`/.snapshots`. The root account was locked at the emergency prompt. Recovery
was performed non-destructively:

1. reboot to GRUB;
2. edit the normal Fedora entry for one boot;
3. add this argument to the kernel line:

   ```text
   rootflags=subvol=root/.snapshots/6/snapshot
   ```

4. boot snapshot 6;
5. mount the shared repository:

   ```bash
   sudo mount -t btrfs \
     -o subvol=root/.snapshots,compress=zstd:1 \
     UUID=806b48e0-565d-4369-9649-dc3ec39c8169 \
     /.snapshots
   ```

6. preserve and inspect snapshot 7;
7. run `sudo tukit rollback 6` to select snapshot 6 as default.

The journal inside snapshot 7 contained only older persistent records; the
failed boot had logged to volatile storage and those records were lost on
reboot. Comparing `/etc/fstab` in snapshots 6 and 7 revealed the typo.

## Current VM state

Snapshot 9 added the corrected entry:

```text
UUID=806b48e0-565d-4369-9649-dc3ec39c8169 /.snapshots btrfs subvol=root/.snapshots,compress=zstd:1 0 0
```

It booted normally and mounted the shared repository automatically. From that
state, this ordinary transaction created snapshot 10:

```bash
sudo tukit execute /usr/bin/true
```

A normal reboot selected snapshot 10 without a GRUB edit. The verified current
mounts are:

```text
/           /dev/vda3[/root/.snapshots/10/snapshot]  subvolid=269
/.snapshots /dev/vda3[/root/.snapshots]              subvolid=258
```

Tukit reports snapshot 10 as both active and default. Snapshot 7 remains
available as evidence of the failed configuration. Persistent journald storage
for read-only transactional roots remains unresolved.

The powered-off `fresh-f44-before-snapper` checkpoint remains the broader
fallback.
