# Tornatus architecture notes

These notes record observations from the Milestone 0 Fedora 44 VM experiment.
They are not a final filesystem layout or an installation specification.

## Milestone 0 question

Can an otherwise conventional Fedora installation use Btrfs, Snapper, and
tukit to:

1. prepare a change in a new root snapshot;
2. leave the running root unchanged;
3. activate the new state on reboot; and
4. roll back to the previous state?

The core transaction, rollback, and repeatable steady-state behavior has been
demonstrated. Several Fedora integration requirements and a persistent logging
gap were also identified.

## Test environment

The observations below came from a real VM, not from a proposed design:

- Fedora Server 44, installed as **Fedora Custom Operating System**
- QEMU/KVM with UEFI, virtio devices, 4 vCPU, 4 GiB RAM, and a 32 GiB disk
- Fedora installer-generated GPT partitioning:
  - 600 MiB vfat EFI system partition
  - 2 GiB XFS `/boot`
  - remaining space as Btrfs
- Fedora installer-generated Btrfs root subvolume named `root`

The initial root mount was:

```text
/dev/vda3[/root] on /
FSROOT=/root
subvolid=256
```

The initial Btrfs default was top-level ID 5 (`FS_TREE`), not the Fedora root
subvolume.

## Fedora package observations

Fedora 44 supplied the required Milestone 0 components directly:

```text
snapper                       0.13.0-3.fc44
tukit                         6.0.0-1.fc44
libtukit                      6.0.0-1.fc44
dracut-transactional-update   6.0.0-1.fc44
```

The installed transaction set also pulled in the required Btrfs and Boost
libraries. The optional `dnf5-plugin-txnupd` package was available but was not
installed for this experiment.

Fedora did not provide a separate package named `tukit-snapper-plugin`. The
`tukit` package itself installed:

```text
/usr/libexec/snapper/plugins/50-etc
```

This corrects the kickoff brief's initial package-list assumption.

Installing the packages did not create a Snapper configuration. Before
configuration, `tukit snapshots` failed because it expects a Snapper config
named `root`.

## Snapper initialization and tukit bootstrap

The root config was created with:

```bash
sudo snapper -c root create-config /
```

Observed result:

- Snapper config `root` targets `/`.
- Snapper created nested subvolume `root/.snapshots`, Btrfs ID 258.
- `/.snapshots` was not a separate mount at this point.
- Snapper listed virtual snapshot `0`, described as `current`.

Snapshot `0` is listable but is not an on-disk snapshot. Both of these tukit
attempts failed:

```text
tukit execute /usr/bin/true
  -> Couldn't determine current snapshot number

tukit --continue=0 -- execute /usr/bin/true
  -> Base snapshot '0' does not exist
```

The initial on-disk base had to be created explicitly:

```bash
sudo snapper -c root create \
  --read-only \
  --description "tornatus initial base" \
  --print-number
```

This produced Snapper snapshot 1, Btrfs subvolume ID 259, with `ro=true`.

Using snapshot 1 as a tukit base still failed while the Btrfs default was ID
5. Tukit successfully cloned and executed the command, but discarded the new
snapshot during finalization because it could not map the existing Btrfs
default to a Snapper snapshot number.

After setting snapshot 1 as the Btrfs default, the same operation succeeded:

```bash
sudo btrfs subvolume set-default 259 /
sudo tukit --continue=1 -- execute /usr/bin/true
```

This created snapshot 2 and selected it as the new Btrfs default. The observed
bootstrap requirement is therefore:

1. create a real initial Snapper snapshot;
2. make that snapshot the Btrfs default; and
3. use it as the first tukit transaction base.

This is an experimental bootstrap sequence, not yet a proposed installer
implementation.

## Boot selection on stock Fedora

Stock Fedora did not initially boot the Btrfs default. Both the installed boot
entry and `/etc/fstab` named the original `root` subvolume. The kernel command
line included:

```text
root=UUID=806b48e0-565d-4369-9649-dc3ec39c8169
rootflags=subvol=root
```

After tukit made snapshot 2 the Btrfs default, a normal reboot still mounted:

```text
/dev/vda3[/root]
FSROOT=/root
subvolid=256
```

Removing only `rootflags=subvol=root` in the GRUB editor for one boot caused
Fedora to mount the Btrfs default successfully:

```text
/dev/vda3[/root/.snapshots/2/snapshot]
FSROOT=/root/.snapshots/2/snapshot
```

The temporary edit was repeated successfully for snapshots 3 and 4.

The persistent current-kernel boot entry was later changed with `grubby`.
`grubby` updated the separate `/boot` entry before failing to update the
read-only `/etc/kernel/cmdline`. The active boot entry then showed:

```text
args="ro rhgb quiet"
```

`/etc/kernel/cmdline` was corrected inside a subsequent tukit transaction.
After activating that snapshot, a completely normal boot selected default
snapshot 6 without manual GRUB editing. `/proc/cmdline` no longer contained a
`rootflags` argument.

Observed conclusion: tukit's Btrfs-default activation model works on Fedora,
but the installer-generated `rootflags=subvol=root` overrides it and must not
remain in the effective boot configuration.

## Snapshot repository mount

Btrfs snapshots are not recursive. The `root/.snapshots` nested subvolume was
therefore not included when a root snapshot became `/`.

After booting snapshot 2, `/.snapshots` was only an ordinary directory inside
that root snapshot. Tukit then failed with:

```text
IO Error (.snapshots is not a btrfs subvolume)
```

Temporarily mounting the original repository subvolume fixed Snapper and tukit:

```bash
sudo mount -o subvolid=258 /dev/vda3 /.snapshots
```

The stable named-subvolume form was also tested successfully:

```bash
sudo mount -t btrfs \
  -o subvol=root/.snapshots,compress=zstd:1 \
  UUID=806b48e0-565d-4369-9649-dc3ec39c8169 \
  /.snapshots
```

With this mount present, tukit correctly reported a running numbered snapshot
as `active=yes` and could perform ordinary transactions without an explicit
`--continue` base.

A first attempt to add a matching `/etc/fstab` entry in snapshot 7 contained a
transcription error:

```text
UUID=806b48e0-565d-4369-9649-dc3ec39c8169 ./snapshots btrfs subvol=root/.snapshots,compress=zstd:1 0 0
```

The target was `./snapshots`, not the intended absolute hidden path
`/.snapshots`. Booting snapshot 7 therefore entered emergency mode. Because the
root account was locked, recovery used a one-time GRUB kernel argument to boot
known-good snapshot 6 directly:

```text
rootflags=subvol=root/.snapshots/6/snapshot
```

After mounting the shared repository, `tukit rollback 6` restored snapshot 6
as the default while preserving failed snapshot 7 for inspection.

Snapshot 9 then added the corrected entry transactionally:

```text
UUID=806b48e0-565d-4369-9649-dc3ec39c8169 /.snapshots btrfs subvol=root/.snapshots,compress=zstd:1 0 0
```

A normal boot into snapshot 9 succeeded. `findmnt` confirmed that `/` used
`/root/.snapshots/9/snapshot` (subvolume ID 268) while `/.snapshots` was a
separate mount of `/root/.snapshots` (subvolume ID 258). Tukit then recognized
snapshot 9 as both active and default without a manual repository mount.

## Transaction isolation and activation

Once a numbered snapshot was active and `/.snapshots` was mounted, the normal
steady-state workflow worked:

```bash
sudo tukit execute /usr/bin/true
```

Tukit inferred active snapshot 2, created snapshot 3, and selected snapshot 3
as default. Before reboot, tukit reported snapshot 2 as active and snapshot 3
as default.

An observable `/etc` change was then tested:

```bash
sudo tukit execute /usr/bin/touch /etc/tornatus-m0-transaction
```

Observed behavior:

1. Tukit created snapshot 4 and made it default.
2. The marker did not exist in still-running snapshot 3.
3. The marker did exist at
   `/.snapshots/4/snapshot/etc/tornatus-m0-transaction`.
4. Booting snapshot 4 made the marker visible at its normal `/etc` path.

This demonstrates isolation of a change from the running system followed by
activation on reboot.

## Repeatable steady-state transaction

With snapshot 9 active and the corrected repository mount supplied by
`/etc/fstab`, a further no-op transaction required no bootstrap options or
manual mount:

```bash
sudo tukit execute /usr/bin/true
```

Tukit used snapshot 9 as its base, created snapshot 10, and selected snapshot
10 as default. A normal reboot reached the login prompt without a GRUB edit.
Post-boot checks showed:

```text
/           /dev/vda3[/root/.snapshots/10/snapshot]  subvolid=269
/.snapshots /dev/vda3[/root/.snapshots]              subvolid=258
```

Tukit reported snapshot 10 as both `active=yes` and `default=yes`. This proves
the complete steady-state loop: transact, select the new default, reboot into
it, retain access to the shared repository, and transact again.

## Rollback

While snapshot 4 was active, this selected snapshot 3 as the new default:

```bash
sudo tukit rollback 3
```

Before reboot, tukit reported snapshot 4 as active and snapshot 3 as default.
After booting the default, the system mounted snapshot 3 and the marker created
only in snapshot 4 was absent.

This satisfies the essential Milestone 0 forward-change and rollback proof.

## Read-only active roots

Tukit-finalized snapshots had the Btrfs property:

```text
ro=true
```

`findmnt` still displayed the root mount as `rw`, but direct writes failed with
`Read-only file system`. Changes to files such as `/etc/kernel/cmdline` and
`/etc/fstab` therefore had to be made through a new tukit transaction.

This distinction matters for diagnostics: mount flags alone do not establish
whether the active root accepts writes.

## Automatic Snapper snapshots

`snapper create-config /` used Fedora's default Snapper settings, including:

```text
TIMELINE_CREATE=yes
TIMELINE_CLEANUP=yes
NUMBER_CLEANUP=yes
```

During the experiment, Snapper automatically created snapshot 5 with
description `timeline`. This was initially noticed immediately after a tukit
rollback, but was not created by the rollback itself.

Systemd state showed:

```text
snapper-timeline.timer  enabled
snapper-cleanup.timer   disabled
```

Thus Fedora's observed default creates hourly timeline snapshots while the
periodic cleanup timer is disabled, despite cleanup policies being present in
the Snapper config. Tornatus will need to make an explicit policy decision
later; no policy change was made during this experiment.

## Failed-boot journal inspection

Snapshot 7 contained a persistent journal directory, but its journal only held
older entries through the initial snapshot creation. The emergency boot itself
was not present after reboot. The read-only transactional roots had fallen back
to volatile journal storage, so the relevant records disappeared when the VM
rebooted.

The `fstab` error was instead established by comparing the preserved failed
snapshot with the active root:

```bash
sudo diff -u /etc/fstab /.snapshots/7/snapshot/etc/fstab
```

This preserved-snapshot comparison was sufficient to expose `./snapshots` as
the cause. Persistent journald storage for read-only roots remains an
integration problem to solve.

## Proven and unresolved

Proven on this Fedora 44 VM:

- Fedora's packaged Snapper and tukit can create transactional root snapshots.
- A transaction does not alter the active root.
- The Btrfs default can select the next root when Fedora does not pin a
  particular subvolume in the kernel command line.
- Reboot activates the new root.
- Tukit rollback selects an earlier root, and reboot activates it.
- Active transactional roots are Btrfs read-only.
- A corrected `/.snapshots` `fstab` entry mounts the shared repository during
  boot.
- The transaction and normal-boot cycle repeats without manual GRUB or mount
  intervention.

Observed integration requirements:

- tukit needs a real numbered initial snapshot and a default that maps to it;
- the boot path must honor the Btrfs default;
- the shared Snapper repository must be available at `/.snapshots` in every
  root snapshot;
- mutations to image-owned system configuration must themselves be
  transactional.

Still unresolved:

- persistent journald storage across failed boots of read-only roots;
- whether the installer-generated root `fstab` entry should be changed;
- the desired automatic snapshot and cleanup policy;
- final persistent-state boundaries for `/etc`, `/var`, and `/home`.
