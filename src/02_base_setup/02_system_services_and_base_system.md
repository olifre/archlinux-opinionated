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
You may want to edit `/etc/xdg/reflector/reflector.conf` to contain your country, e.g.:
```
--country Germany
```

## Install some basic tools
Some tools from Arch repos:
```
pacman -S powertop guvcview chromium firefox firefox-i18n-de thunderbird thunderbird-i18n-de nextcloud-client fwupd stress-ng mpv libreoffice-fresh libreoffice-fresh-de power-profiles-daemon keepassxc wl-clipboard xclip waypipe rsync biber python-pygments xorg-xlsclients inkscape screen strace iftop iotop-c htop tcpdump compsize scrcpy emacs-wayland wireshark-qt tcpdump gimp speedtest-cli iperf3 freerdp wakeonlan github-cli fortune-mod syncthing zathura zathura-pdf-poppler zathura-ps zathura-cb usbutils arandr jq yq wev yubikey-personalization-gui yubikey-manager root jupyter-metakernel gnuplot python-matplotlib python-numpy python-pandas python-scipy pv python-pip perf tigervnc networkmanager-openconnect bind hid-tools sshpass ethtool ndisc6 xrootd kdiff3 apptainer diffpdf diffoscope mdbook hugo wget aria2 python-jinja 7zip cpupower rebuild-detector obs-studio labplot pdftk qpdf tesseract-data-eng progress xournalpp mokutil
```
Then, the groups:
```
pacman -S texlive
```
and also the package:
```
pacman -S texlive-langgerman
```
and from AUR:
```
yay -S syncthingtray-qt6 powerstat afc charliecloud
```

## Configure `nano`
Edit `/etc/nanorc`, set:
```
set cutfromcursor
```

## Install the desktop environment with apps
```
yay -S plasma-meta kde-applications-meta
```
Then, execute:
```
systemctl enable --now plasmalogin
```

## Configure SDDM (not used anymore!)
> [!WARNING]
> I have since migrated to `plasma-login-manager`, activated above. So this is not needed anymore, It uses wayland and runs rootless out of the box.

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
You might want to set the theme `bgrt` which is the ArchLinux default in any case, as can be confirmed with:
```
plymouth-set-default-theme
```
On an older system on which BGRT does not receive an image from UEFI, another interesting theme could e.g. be the `breeze` theme provided by the `breeze-plymouth` package.

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
# better compression when activated for a logfile pattern
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
    minsize 10M
    compress
	dateext
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
    minsize 10M
    compress
	dateext
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

## Activate monthly BTRFS scrub
A monthly scrub can highlight inconsistencies early on:
```
systemctl enable --now btrfs-scrub@-.timer
```
The `-` is the `systemd-escape` variant of the `/` filesystem.
Results of the scrubs can then be found in the system journal:
```
journalctl -u btrfs-scrub@-.service
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

## BEES (for btrfs dedupe) {#beesd}
Install with:
```
yay -S bees
```
Then copy over config:
```
cp /etc/bees/beesd.conf.sample /etc/bees/beesd_root.conf
```
and adapt it, set `UUID` to the `UUID` returned for `lsblk -f`. 
You should also set:
```
OPTIONS="-v 6"
```
for reduced verbosity, and you also might want to make the `DB_SIZE` default setting explicit:
```
DB_SIZE=$((1024*1024*1024)) # 1G in bytes
```

If you already have many `btrbk` snapshots, you may want to reduce the number of snapshots first.

Finally, start the service using the `UUID`, for example:
```
systemctl enable --now beesd@b8a34ebc-029a-4c77-ac2c-33290c18b461.service
```
Check the `journal` on progress, and also `/var/run/bees` contains status information. 

Note that after the first completed `bees` run, you might want to make sure to remove old snapshots from pre-`bees` to ensure they do not remain with duplicated data.

You might also want to check out statistics in `/mnt/btrfs_pool/.beeshome` (note that `/mnt/btrfs_pool` is not mounted by default). 

## Set up `fwupd` for secure boot
Sign it once manually:
```
sbsign --key /etc/refind.d/keys/refind_local.key --cert /etc/refind.d/keys/refind_local.crt /usr/lib/fwupd/efi/fwupdx64.efi
```
and then create the needed hook, create the file `/etc/pacman.d/hooks/sign-fwupd-secureboot.hook` with content:
```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/fwupd/efi/fwupdx64.efi

[Action]
When = PostTransaction
Exec = /usr/bin/sbsign --key /etc/refind.d/keys/refind_local.key --cert /etc/refind.d/keys/refind_local.crt /usr/lib/fwupd/efi/fwupdx64.efi
Depends = sbsigntools
```
See also [this ArchWiki article](https://wiki.archlinux.org/title/Fwupd#Secure_Boot), note we must leav `shim` usage active as we are using a MOK.

For `fwupd` to work, you also need a directory in your ESP which can be used:
```
mkdir -p /efi/EFI/arch
mkdir -p /efi/EFI/systemd
```
Note that if both directories are present, the `systemd` one will be used preferredly.
You must also deploy `shim-signed` there, which was prepared with a hook earlier. You can trigger this manually either be reinstalling `shim-signed` or by copying it manually:
```
cp /usr/share/shim-signed/shimx64.efi /efi/EFI/arch/shimx64.efi
cp /usr/share/shim-signed/shimx64.efi /efi/EFI/systemd/shimx64.efi
```

To make this look nicer in the `refind` menu, which would assume the `arch` icon or a generic one otherwise, copy over an icon to the directory:
```
cp /efi/EFI/refind/icons/tool_fwupdate.png /efi/EFI/arch/fwupdx64.png
cp /efi/EFI/refind/icons/tool_fwupdate.png /efi/EFI/systemd/fwupdx64.png
```
