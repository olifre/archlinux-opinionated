# System services and base system installation

## Set up NTP
```
systemctl enable --now systemd-timesyncd.service
```

## Set up some base services
These are ArchLinux-specific.
```
pacman -S reflector
pacman -S pacman-contrib
systemctl enable --now paccache.timer
systemctl enable --now fstrim.timer
systemctl enable --now reflector.timer
```

## Install some basic tools
Some tools from Arch repos:
```
pacman -S powertop guvcview chromium firefox thunderbird nextcloud-client fwupd stress-ng mpv libreoffice-fresh power-profiles-daemon keepassxc wl-clipboard xclip waypipe rsync biber python-pygments xorg-xlsclients inkscape screen strace iftop iotop-c htop tcpdump compsize scrcpy emacs-wayland wireshark-qt tcpdump gimp speedtest-cli iperf3 freerdp wakeonlan github-cli fortune-mod syncthing zathura zathura-pdf-poppler zathura-ps zathura-cb usbutils arandr jq yq wev
```
Then, the groups:
```
pacman -S texlive
```
and from AUR:
```
yay -S syncthingtray-qt6 powerstat
```

## Install the desktop environment with apps
```
yay -S plasma-meta kde-applications-meta sddm
```
Then, execute:
```
systemctl enable --now sddm
```

## Configure SDDM
Set it up to use `wayland` (rootless):
```
mkdir /etc/sddm.conf.d/
cd /etc/sddm.conf.d/
```
Create `/etc/sddm.conf.d/05-base.conf` with content:
```
[Theme]
# Current theme name
Current=breeze

# Cursor theme used in the greeter
CursorTheme=breeze_cursors
```
Create `10-wayland.conf` with content:
```
[General]
DisplayServer=wayland
GreeterEnvironment=QT_WAYLAND_SHELL_INTEGRATION=layer-shell

[Wayland]
CompositorCommand=kwin_wayland --drm --no-lockscreen --no-global-shortcuts --locale1
```
Finally, restart it:
```
systemctl restart sddm.service
```

## Set up Plymouth
```
yay -S plymouth plymouth-kcm
```
Then, in `/etc/mkinitcpio.conf`, add `plymouth` to `HOOKS` after `systemd`, but before `sd-encrypt`, then:
```
mkinitcpio -P
```

## Set up `firewalld`
Install `firewalld`, `firewall-applet` and `firewall-config`:
```
yay -S firewalld firewall-applet firewall-config
```
Note that the integration into `KDE` is not too helpful at this point, it does not support zones. 
After installation, activate:
```
systemctl enable --now firewalld
```
For further configuration, you can start `firewall-config` (GUI) and allow `syncthing`, `kdeconnect`. Alternatively, you can run:
```
firewall-cmd --zone=public --add-service syncthing
firewall-cmd --zone=public --add-service kdeconnect
firewall-cmd --runtime-to-permanent
```
Note `ssh` and `dhcpv6-client` are already on by default, see:
```
firewall-cmd --info-zone=public
```

## Set up `zram`
Install `zram-generator`:
```
yay -S zram-generator
```
then edit `/etc/systemd/zram-generator.conf`, should contain (swap and personal scratch space):
```
[zram0]
zram-size = min(ram / 2, 16384)
compression-algorithm = zstd

[zram1]
zram-size = min(ram / 2, 16384)
mount-point = /var/tmp/olifre
options = X-mount.owner=1000,X-mount.group=1000
```
Create `/etc/sysctl.d/99-vm-zram-parameters.conf` with content:
```
vm.swappiness = 180
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0
```

## Set up `locate`
Install package `plocate`:
```
yay -S plocate
```
Edit `/etc/updatedb.conf` and set (to include `btrfs` filesystems):
```
PRUNE_BIND_MOUNTS = "no"
```
You may want to enable the timer (but also happens on reboot) or trigger the service for an initial indexing:
```
systemctl start plocate-updatedb.timer
systemctl start plocate-updatedb.service
```

## Set up `logrotate`
Install package `logrotate`:
```
yay -S logrotate
```
Edit `/etc/logrotate.conf` (uncomment / add as-needed):
```
# use date as a suffix of the rotated file.
dateext

# compress rotated log files.
compress

# better compression
compresscmd /usr/bin/xz
uncompresscmd /usr/bin/xz
compressext .xz
compressoptions "-9"

notifempty
```
Note we already set up things here for the backup we'll set up later.
Create file `/etc/logrotate.d/restic` with content:
```
/var/log/restic/*.log {
    weekly
    missingok
    rotate 100
    copytruncate
    minsize 1M
    compress
}
```
You'll also want to create this:
```
mkdir /var/log/restic
```
Create file `/etc/logrotate.d/btrbk` with content:
```
/var/log/btrbk.log {
    weekly
    missingok
    rotate 100
    copytruncate
    minsize 1M
    compress
}
```
Enable timer and trigger once:
```
systemctl enable --now logrotate.timer
systemctl start logrotate
```

## Set up `cronie`
```
systemctl enable --now cronie
```

## Set up `dnsmasq`
Install package `dnsmasq`:
```
yay -S dnsmasq
```
then, edit `/etc/NetworkManager/NetworkManager.conf` and add:
```
[main]
dns=dnsmasq
```
For more safe and easy usage of VPNs, you may want to create `/etc/NetworkManager/dnsmasq.d/fritzbox` with content:
```
server=/fritz.box/192.168.22.1
```
(assuming this is your home router hostname and IP).
Finally, apply:
```
systemctl restart NetworkManager
```

## Tinc VPN
Install `tinc-pre` (from AUR):
```
yay -S tinc-pre
```
Execute:
```
tinc -n homeroute init myhostname
```
Note that this does 2048 RSA, we want 4096, so:
```
tinc -n homeroute generate-keys 4096
```
Now, clean out the old keys, i.e. the commented parts of:
- `/etc/tinc/homeroute/{ed25519_key,rsa_key}.priv`
- `/etc/tinc/homeroute/hosts/myhostname`

Copy over config parts from existing tinc cluster, i.e. up/down scripts, other hosts, `tinc.conf` parts.
If you use static addressing, do not forget to adapt IPs in up/down scripts and add a static `Address` to this host's config!
Finally, copy over the host config file to all other nodes.

## Set up Bluetooth
```
systemctl enable --now bluetooth.service
```

## Set up hardware acceleration and similar
Install packages:
```
yay -S vulkan-intel vulkan-mesa-layers intel-media-driver libva-utils
```
Check things work:
```
vainfo
vulkaninfo
```
