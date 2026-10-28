# Runbook: Move the homeserver to the 256 GB SSD

Tracker #40. The new SSD replaces the OCZ Vertex 3 (56 GB, 2011) as the
system disk and then also holds the k3s VM disks, which today sit on the
black HDD and saturate it on every boot (#12: ~17 min until the apps
answer after a reboot). No encryption in this move (decision in #40,
encryption is #41).

The method is a **block-level clone** of the OCZ, then growing root into
the larger disk. The clone keeps every UUID and partition GUID, so
`/etc/fstab` and the UEFI boot entry stay valid without edits. The OCZ
is kept unchanged as the rollback.

Current layout (2026-10-28):

| Partition | Size | Type | UUID (filesystem) |
| --- | --- | --- | --- |
| OCZ part 1 | 976 M | vfat, `/boot/efi` | `297B-3B35` |
| OCZ part 2 | 52.1 G | ext4, `/` | `ec507031-9918-4b9c-9610-7a36c67ba83c` |
| OCZ part 3 | 2.9 G | swap | `43175eab-35f9-44c2-8af7-a7a97675c9e1` |

Device names (`sda`/`sdb`/`sdc`) change between boots. Always identify
disks by **model and size** (`lsblk -o NAME,SIZE,MODEL,TRAN`): the OCZ
is `OCZ-VERTEX3` (55.9 G), the HDDs are `WDC WD1003FZEX` (black, 931 G)
and `WDC WD20EFRX` (red, 1.8 T).

## Phase 0: before the downtime

1. **SMART of the new SSD**, attached over the USB-SATA adapter (to the
   laptop is fine): `sudo smartctl -a -d sat /dev/sdX`. Expect `PASSED`,
   0 reallocated/pending sectors, near-zero power-on hours.
2. **Live USB stick**: Debian 13 live (standard, no desktop needed). It
   has `dd`, `parted`, `e2fsprogs` and `util-linux` (`mkswap`).
3. `home-infra#104` merged (VM disk move support in `k3s_node`) and
   `scripts/roll-out-k3s.sh dry-run` showing no changes.
4. **Right before the downtime:** `scripts/verify-backups.sh` with all
   checks `PASS` (see `verify-backups.md`).

## Phase 1: clone (downtime, at the homeserver)

1. From the laptop: `scripts/shutdown-homeserver-for-maintenance.sh`
   (README, "Hardware maintenance shutdown"). It stops the VMs and
   Nextcloud AIO cleanly and powers off.
2. Attach the new SSD via the USB-SATA adapter, boot the live stick.
3. Identify both disks and write the names down:
   ```bash
   lsblk -o NAME,SIZE,MODEL,TRAN
   ```
   `OLD` = the `OCZ-VERTEX3`, `NEW` = the 256 GB disk on `usb`. **A swap
   of the two destroys the system**: check twice.
4. Clone (~56 GB, a few minutes over USB 3):
   ```bash
   sudo dd if=/dev/OLD of=/dev/NEW bs=16M conv=fsync status=progress
   ```
5. Grow root. The swap partition sits behind root, so it is recreated at
   the end of the disk **with its old UUID** (fstab stays as is):
   ```bash
   sudo parted /dev/NEW print            # answer "Fix" to move the backup GPT to the end
   sudo parted /dev/NEW rm 3
   sudo parted /dev/NEW resizepart 2 -3GiB
   sudo parted /dev/NEW mkpart swap linux-swap -3GiB 100%
   sudo e2fsck -f /dev/NEW2
   sudo resize2fs /dev/NEW2
   sudo mkswap -U 43175eab-35f9-44c2-8af7-a7a97675c9e1 /dev/NEW3
   lsblk -o NAME,SIZE,FSTYPE,UUID /dev/NEW   # UUIDs as in the table above, root ~235 G
   ```
6. Power off. Swap the disks inside: the new SSD on the OCZ's SATA port,
   the OCZ out. **Never boot with both attached**: same UUIDs and
   partition GUIDs, the system could mount parts of either. Label the
   OCZ and keep it unchanged.

## Phase 2: first boot

Boot normally (the UEFI entry `debian` points at the ESP's partition
GUID, which the clone kept). Then from the laptop:

```bash
ssh julian@192.168.178.100 'lsblk -o NAME,SIZE,MODEL,MOUNTPOINTS; df -h /; swapon --show; systemctl --failed'
```

Expect `/` on the new disk at ~235 G, swap active, no failed units. Then:
Nextcloud AIO containers up, NFS exports (`showmount -e localhost`),
`virsh list --all` (both VMs running, disks still on the HDD at this
point), k3s nodes `Ready`, and `site.yml --check` with changes only
where expected.

## Phase 3: move the VM disks (short k3s downtime)

A `home-infra` PR sets, in `ansible/playbooks/k3s.yml` for both nodes:

```yaml
k3s_node_storage_mount: /
k3s_node_vm_storage_dir: /var/lib/libvirt/k3s
k3s_node_vm_disk_path: /var/lib/libvirt/k3s/<node>.qcow2
k3s_node_vm_previous_disk_path: /mnt/black-hdd/k3s/<node>.qcow2
```

Then `scripts/roll-out-k3s.sh` (dry-run first). Per node, the role stops
the VM, copies the disk sparse (node-1 ~23 G, node-2 ~44 G), redefines
the domain and starts it. Expect a few minutes per node. Space: the OS
uses ~23 G, the VM disks ~67 G today and at most 140 G at their virtual
sizes, all inside ~235 G.

The old disks stay on the HDD as the rollback for about a week, then
get deleted by hand and `k3s_node_vm_previous_disk_path` is removed.

## Phase 4: proof

- Reboot test as in #34 (watch SSH and the public routes from the
  laptop): apps answering clearly sooner than the ~17 min of 2026-10-28.
- `scripts/verify-backups.sh` again.
- Update `current-state.md` (system disk, VM disk location) and close
  #40, #12.

## Rollback

- **OS:** power off, put the OCZ back on its port (new SSD out), boot.
  Everything is as before the move.
- **VMs after phase 3:** the same move in reverse. Point the variables
  back at the HDD and set `k3s_node_vm_previous_disk_path` to the SSD
  path. The HDD disk still exists, so the copy is skipped and the domain
  is redefined on it: the node comes back with its state from before the
  move.

Disposing of the OCZ later: it holds every credential of the old system
unencrypted, so **ATA Secure Erase** first.
