# System services and base system installation

## Set up some base services
```
pacman -S reflector
pacman -S pacman-contrib
systemctl enable --now systemd-timesyncd.service
systemctl enable --now paccache.timer
systemctl enable --now fstrim.timer
systemctl enable --now reflector.timer
```

## Install some basic tools
Some tools from Arch repos:
```
powertop
guvcview
chromium
firefox
thunderbird
nextcloud-client
fwupd
stress-ng
mpv
libreoffice-fresh
power-profiles-daemon
keepassxc
wl-clipboard
xclip
waypipe
rsync
biber
python-pygments
xorg-xlsclients
inkscape
screen
strace
iftop
iotop-c
htop
tcpdump
compsize
scrcpy
emacs
wireshark-qt
tcpdump
gimp
```
Then, the groups:
```
texlive
```
and from AUR:
```
syncthingtray-qt6
```

## Install the desktop environment with apps
```
plasma-meta
kde-applications-meta
sddm
systemctl enable --now sddm
```
