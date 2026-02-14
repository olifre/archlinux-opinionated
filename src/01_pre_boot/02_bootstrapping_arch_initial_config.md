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

## Prepare the network setup
Edit `/etc/hostname`, set:
```
myhostname-i-have-though-about
```
Then, enable `NetworkManager`:
```
systemctl enable NetworkManager
```
In case you need `eduroam`, remember to install `python-dbus` so the CAT tool works:
```
pacman -S python-dbus
```
For IPv6 with working DNS with most home routers, you may need:
```
pacman -S avahi-daemon
systemctl enable avahi-daemon
```
You may want to enable privacy extensions by creating `/etc/sysctl.d/ipv6-priv.conf` with content (adapt to interface names!):
```
# Enable ipv6 privacy extensions
net.ipv6.conf.wlp0s20f3.use_tempaddr = 2
net.ipv6.conf.all.use_tempaddr = 2
net.ipv6.conf.default.use_tempaddr = 2
```
You might also want to install `dhcpcd`:
```
pacman -S dhcpcd
```
and tell NetworkManager to use it instead of it's integrated implementation by adding these lines to `/etc/NetworkManager/NetworkManager.conf`:
```
[main]
dhcp=dhcpcd
```

## Setup `initrd`
Edit `/etc/mkinitcpio.conf`, ensure `sd-encrypt` is added:
```
HOOKS=(base systemd autodetect microcode modconf kms keyboard keymap sd-vconsole block sd-encrypt filesystems fsck)
```
Run `mkinitcpio -P`.

## Set `root` password
```
passwd
```

## Add `refind_linux.conf` for boot configuration
Create the file `/boot/refind_linux.conf` with content:
```
"Boot with standard options no EHT"     "zswap.enabled=0 splash rw iwlwifi.disable_11be=1"
"Boot with standard options"    "zswap.enabled=0 splash rw"
"Boot without plymouth" "zswap.enabled=0 plymouth.enable=0 disablehooks=plymouth rw"
"Boot with full UUIDs"  "rd.luks.name=9f26906e-7603-4735-91e4-81f412f089cc=ArchLinux root=/dev/mapper/ArchLinux zswap.enabled=0 rw"
"Boot to single user mode" "zswap.enabled=0 splash ro single"
```
Note the special `iwlwifi` setting is due to problems with EHT/MLO support in case of my hardware, the UUID of course needs to be adapted, and `zswap` is turned off since we are using `zram` which will be set up later.

## Test booting
At this point, `refind` is not installed yet. As outlined in [Booting from the Installation medium](01_installation_system_partitioning_formatting.md#instmedium), we'll now use a rEFInd boot loader from an external medium, e.g. a [Ventoy](https://www.ventoy.net/) boot stick. You should be able to boot with that by now. 

This should work fine, if yes, you are ready to continue to the next stage!
