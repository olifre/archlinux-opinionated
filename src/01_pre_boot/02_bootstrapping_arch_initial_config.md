# Bootstrapping Arch and initial config

## Initial `pacstrap`
```
pacman-key --init
pacman-key --populate
pacman -Sy archlinux-keyring
pacstrap -K /mnt/rootfs base linux linux-lts sof-firmware linux-firmware intel-ucode wireless-regdb networkmanager nano man-db man-pages texinfo btrfs-progs dosfstools e2fsprogs openssh gdisk
```

## Initial config
Follow usual manual for initial config, basically:
```
genfstab -U /mnt >> /mnt/rootfs/etc/fstab
arch-chroot /mnt/rootfs
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
```

## "Savepoint"
In case you break off here, you can continue by booting from the installation medium again and setting up keys etc. (see [Booting from the Installation medium](01_installation_system_partitioning_formatting.md#instmedium)), then issue:
```
cryptsetup open /dev/nvme0n1p3 ArchLinux
mount --mkdir -t btrfs -o defaults,noatime,compress-force=zstd:6,ssd /dev/mapper/ArchLinux /mnt/rootfs
mount -t btrfs -o defaults,noatime,compress-force=zstd:6,ssd,subvol=home /dev/mapper/ArchLinux /mnt/rootfs/home
mount /dev/nvme0n1p1 /mnt/rootfs/efi
mount /dev/nvme0n1p2 /mnt/rootfs/boot
arch-chroot /mnt/rootfs
```

## Setup locales
In `/etc/locale.gen`, uncomment
```
en_US.UTF-8 UTF-8
de_DE.UTF-8 UTF-8
```
Then, run `locale-gen`.  
Finally, in `/etc/locale.conf`, set:
```
LANG=de_DE.UTF-8
```
and in `/etc/vconsole.conf`, set:
```
KEYMAP=de-latin1
```
