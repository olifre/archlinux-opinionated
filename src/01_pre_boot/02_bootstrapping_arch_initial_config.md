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
In case you break off here, you can continue by booting from the installation medium again and setting up keyboard layout etc. (see [Booting from the Installation medium](01_installation_system_partitioning_formatting.md#instmedium)), then issue:
```
cryptsetup open /dev/nvme0n1p3 ArchLinux
mount --mkdir -t btrfs -o defaults,noatime,compress-force=zstd:6,ssd /dev/mapper/ArchLinux /mnt/rootfs
mount -t btrfs -o defaults,noatime,compress-force=zstd:6,ssd,subvol=home /dev/mapper/ArchLinux /mnt/rootfs/home
mount /dev/nvme0n1p1 /mnt/rootfs/efi
chmod 0750 /mnt/rootfs/efi
mount /dev/nvme0n1p2 /mnt/rootfs/boot
chmod 0750 /mnt/rootfs/boot
arch-chroot /mnt/rootfs
```

## Set up locales
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

> [!WARNING]
> While `dhcpcd` is still a great tool, I do not use it with NetworkManager in this way anymore, i.e. I am using the internal DHCP client. Reasoning: It seems that NetworkManager uses randomized MACs for scanning even while connected, which seems to confuse `dhcpcd` as the MAC changes and it obediently drops the IP address lease due to this, which in turn breaks connectivity.

You might also want to install `dhcpcd`:
```
pacman -S dhcpcd
```
and tell NetworkManager to use it instead of it's integrated implementation by adding these lines to `/etc/NetworkManager/NetworkManager.conf`:
```
[main]
dhcp=dhcpcd
```

## Set up `initrd`
Edit `/etc/mkinitcpio.conf`, ensure `sd-encrypt` is added:
```
HOOKS=(base systemd autodetect microcode modconf kms keyboard keymap sd-vconsole block sd-encrypt filesystems fsck)
```
Run `mkinitcpio -P`.

## Set `root` password
```
passwd
```

## Add `refind_linux.conf` for boot configuration {#refindlinuxconf}
Create the file `/boot/refind_linux.conf` with content:
```
"Boot with standard options"            "zswap.enabled=0 rd.luks.options=tries=10,timeout=0 rootflags=x-systemd.device-timeout=0 quiet splash rw"
"Boot with standard options no EHT"     "zswap.enabled=0 rd.luks.options=tries=10,timeout=0 rootflags=x-systemd.device-timeout=0 quiet splash rw iwlwifi.disable_11be=1"
"Boot without plymouth"                 "zswap.enabled=0 rd.luks.options=tries=10,timeout=0 rootflags=x-systemd.device-timeout=0 plymouth.enable=0 disablehooks=plymouth rw"
"Boot to terminal"                      "zswap.enabled=0 rd.luks.options=tries=10,timeout=0 rootflags=x-systemd.device-timeout=0 rw systemd.unit=multi-user.target"
"Boot with full UUIDs"                  "rd.luks.name=bdf5159d-ad5d-4d1d-bfac-ce833c92048d=ArchLinux root=/dev/mapper/ArchLinux zswap.enabled=0 rd.luks.options=tries=10,timeout=0 rootflags=x-systemd.device-timeout=0 rw"
"Boot to single user mode"              "zswap.enabled=0 rd.luks.options=tries=10,timeout=0 rootflags=x-systemd.device-timeout=0 rw single"
"Boot with minimal options"             "ro"
```
Note the special `iwlwifi` setting is due potential to problems with EHT/MLO support in case of my hardware, the UUID of course needs to be adapted, and `zswap` is turned off since we are using `zram` which will be set up later. The timeout settings for LUKS are important in case the password entry is delayed, as otherwise `systemd` would time out if the password is not entered within 90 seconds and you'd be left with an unbootable system which can only be turned off hard, see [this link to the ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Timeout). We also increase the number of `tries` a bit. Note that in some cases, `sd-encrypt` seems to count attempts twice or may not retry at all unless the UUID is used explicitly.

## Test booting
At this point, `refind` is not installed yet. As outlined in [Booting from the Installation medium](01_installation_system_partitioning_formatting.md#instmedium), we'll now use a rEFInd boot loader from an external medium, e.g. a [Ventoy](https://www.ventoy.net/) boot stick. You should be able to boot with that by now. 

This should work fine, if yes, you are ready to continue to the next stage!
