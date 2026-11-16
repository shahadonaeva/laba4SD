# Lab 4 — Partition, mirror, fail, recover

**Student:** Shakhzoda Donayeva
**Machine:** MacBook Air (M3, 16 GB RAM, 512 GB)(acer nitro VB); hypervisor: `<FILL: UTM / Parallels / VMware Fusion>`; guests: `<FILL: Ubuntu Server version>` and `<FILL: Windows 11 ARM version>`
**Date:** `<FILL>`

## Use of AI (disclosure)
The structure, wording of explanations and the helper script `run-lab04.sh` were prepared with Claude (Anthropic). I ran the script and the commands myself in my own virtual machines; every file in `evidence/` and `screenshots/` comes from my own run. `<FILL: what you checked or changed yourself>`

## What I did

### 0. Extra disks
Added three virtual disks of 2 GB each to the Ubuntu guest (seen as `sdb`, `sdc`, `sdd`) and one 2 GB disk to the Windows guest (Disk 1). Screenshots: `screenshots/<FILL>.png`.

### 1. Windows: partition, format, drive letter
PowerShell as Administrator:

```powershell
$env:firmware_type
Get-Disk
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter
Get-Partition -DiskNumber 1 | Where-Object DriveLetter | Format-Volume -FileSystem NTFS -NewFileSystemLabel LAB04
Get-Disk; Get-Partition -DiskNumber 1
```

**MBR or GPT — GPT.** The guest boots with UEFI, and UEFI Windows 11 boots only from a GPT system disk. For the data disk GPT is also the better default: the partition table is stored twice (start and end of the disk) with checksums, it supports 128 partitions instead of 4 primary ones, and disks larger than 2 TB. MBR is needed only for old BIOS-only systems, which is not the case here.

### 2. Linux: partitions and the RAID 1 mirror
Run `./run-lab04.sh` inside the Ubuntu guest. It executes, step by step:

```bash
sudo parted -s /dev/sdb mklabel gpt mkpart raid 1MiB 100% set 1 raid on   # and the same for sdc
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 --metadata=1.2 --bitmap=internal --run /dev/sdb1 /dev/sdc1
sudo mkfs.ext4 -L mirror /dev/md0
sudo mount /dev/md0 /mnt/mirror
```
The mirror is built from partitions, not raw disks, so each disk carries a label showing what it belongs to.

### 3. Test data and checksum
A small readable file (`proof.txt`) and a 100 MB file from `/dev/urandom`, then `sha256sum` — any changed bit would change the hash.

### 4. Fail one disk
`mdadm --fail` + `--remove` on the second partition, then `vm.drop_caches=3` so the next checksum is read from the remaining disk, not from memory.

### 5. Add the third disk and rebuild
Partition the third disk, `mdadm --add`, watch `/proc/mdstat` until recovery ends, check checksums again.

## Result

### States of the array
| State | `/proc/mdstat` | Evidence |
|---|---|---|
| Healthy | `<FILL from evidence>` | `evidence/array-healthy.txt` |
| Degraded | `<FILL>` | `evidence/array-degraded.txt` |
| Rebuilding | `<FILL>` | `evidence/array-rebuilt.txt` |
| Rebuilt | `<FILL>` | `evidence/array-rebuilt.txt` |

### Checksums (copy from your evidence files)
| File | Before failure | One disk left | After rebuild |
|---|---|---|---|
| proof.txt | `<FILL>` | `<FILL>` | `<FILL>` |
| random-100M.bin | `<FILL>` | `<FILL>` | `<FILL>` |

Conclusion: `<FILL: identical / not identical, and what that shows>`

### Windows layout (`<FILL: evidence/windows-partition.txt>`)
Disk 1 before: `<FILL>`; after: `<FILL>` (partition type, drive letter, file system, label).

## Question: a colleague deleted the wrong file an hour ago — does RAID 1 help?
No. RAID 1 copies every write to both disks immediately, so a deletion is mirrored within moments and the file is gone from both. A mirror protects against a **disk failure**, not against human or software mistakes. For that you need a backup or snapshot made before the deletion and stored where the mistake cannot reach.

## What did not work the first time
`<FILL: describe your real problems and how you solved them — or write "nothing went wrong">`

## Evidence
- `evidence/lsblk-before.txt` — disks before any change
- `evidence/array-healthy.txt` — mirror created and healthy
- `evidence/checksum-before.txt` — SHA-256 before the failure
- `evidence/array-degraded.txt` — one disk failed and removed
- `evidence/checksum-after-failure.txt` — SHA-256 with one disk left
- `evidence/array-rebuilt.txt` — third disk added, rebuild finished, SHA-256 again
- `screenshots/` — VirtualBox/UTM storage settings, terminal outputs, Windows Disk Management
