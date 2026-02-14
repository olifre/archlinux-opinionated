# Installation system, partitioning and formatting

## Booting from installation medium {#instmedium}
Boot from installation iso, set up keymap and font for installation environment:
```
loadkeys de-latin1
setfont ter-132b
```
It's recommended to use a [Ventoy](https://www.ventoy.net/) boot stick, as we'll also rely on using a [rEFInd](https://www.rodsbooks.com/refind/) boot loader later on. For that, you can just put the official refind boot img on the Ventoy stick you have.

## Partitioning
For partition#ing, we use `gdisk`:
```
# gdisk /dev/nvme0n1
```
1. Delete all partitions by creating a new GPT partition table with `o`.
2. Then create partitions with this layout:
   ```
   Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2099199   1024.0 MiB  EF00  EFI system partition
   2         2099200         4196351   1024.0 MiB  EA00  XBOOTLDR partition
   3         4196352      3907028991   1.8 TiB     8304  ArchLinux-Crypt
   ```
3. Note new partitions can be created with `n`, printing works with `p`, and `c` allows to rename a partition.
4. Finally, write out the table with `w`.
5. Note that 8304 is required even for crypted volume to allow for GPT automounting.

## Formatting disks
```
mkfs.fat -F 32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
# Takes good defaults into account, may want to check sector size via https://wiki.archlinux.org/title/Advanced_Format#NVMe_solid_state_drives ...
cryptsetup luksFormat -s 512 /dev/nvme0n1p3
# Check with:
cryptsetup luksDump /dev/nvme0n1p3
cryptsetup open /dev/nvme0n1p3 ArchLinux
mkfs.btrfs -O block-group-tree /dev/mapper/ArchLinux
```

## Setup BTRFS
Setup BTRFS with separate volumes for `rootfs` and `home` for usage of `btrbk`:
```
mount --mkdir -t btrfs -o defaults,noatime,compress-force=zstd:6,ssd /dev/mapper/ArchLinux /mnt/btrfs_pool
btrfs subvol create /mnt/btrfs_pool/rootfs
btrfs subvol create /mnt/btrfs_pool/home
mkdir /mnt/btrfs_pool/_btrbk_snap
mount --mkdir -t btrfs -o defaults,noatime,compress-force=zstd:6,ssd,subvol=rootfs /dev/mapper/ArchLinux /mnt/rootfs
btrfs subvolume set-default /mnt/rootfs
mount --mkdir -t btrfs -o defaults,noatime,compress-force=zstd:6,ssd,subvol=home /dev/mapper/ArchLinux /mnt/rootfs/home
btrfs subvolume list /mnt/btrfs_pool
mount --mkdir /dev/nvme0n1p1 /mnt/rootfs/efi
mount --mkdir /dev/nvme0n1p2 /mnt/rootfs/boot
```
