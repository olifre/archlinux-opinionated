# Power Saving

## Increase dirty writeback
Note this may lead to data loss up to 60 seconds. We have already adapted the `commit` time in `/etc/fstab` earlier.

Create `/etc/sysctl.d/dirty.conf` with content:
```
vm.dirty_writeback_centisecs = 6000
```
You may want to run `mkinitcpio -P` afterwards. 

## Enable WiFi power-save
Create `/etc/modprobe.d/iwlwifi.conf` with content:
```
options iwlwifi power_save=1
```
You may want to run `mkinitcpio -P` afterwards. 

## Activate PCI Runtime Power Management
Create `/etc/udev/rules.d/pci_pm.rules` with content:
```
SUBSYSTEM=="pci", ATTR{power/control}="auto"
SUBSYSTEM=="ata_port", KERNEL=="ata*", ATTR{device/power/control}="auto"
```
You may want to run `mkinitcpio -P` afterwards. 

## Activate USB power saving 
Create `/etc/udev/rules.d/50-usb_power_save.rules` with content:
```
# blacklist for usb autosuspend
# ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="05c6", ATTR{idProduct}=="9205", GOTO="power_usb_rules_end"

# CHERRY G83 (RS 6000) Keyboard
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="046a", ATTR{idProduct}=="0011", GOTO="power_usb_rules_end"

# Dell Computer Corp. Dell MS116 Optical Mouse
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="413c", ATTR{idProduct}=="301a", GOTO="power_usb_rules_end"

ACTION=="add", SUBSYSTEM=="usb", TEST=="power/control", ATTR{power/control}="auto"
LABEL="power_usb_rules_end"
```
Note that the `blacklist` can be used in case you encounter a device which has problems with that. 
You may want to run `mkinitcpio -P` afterwards. 

## Setup some services to run only on AC power
This requires some preparation, as it is not a built-in feature to `systemd` or `udev`.

Create the file `/etc/systemd/system/battery-power.target` with content:
```
[Unit]
Description=Battery-only services
```
and the file `/etc/systemd/system/ac-power.target` with content:
```
[Unit]
Description=AC-only services
```
Then, hook that unit `udev` by creating `/etc/udev/rules.d/99-powertargets.rules` with content:
```
SUBSYSTEM=="power_supply", KERNEL=="AC", ATTR{online}=="0", RUN+="/usr/bin/sh -c '/usr/sbin/systemctl stop ac-power.target && /usr/sbin/systemctl start battery-power.target'"
SUBSYSTEM=="power_supply", KERNEL=="AC", ATTR{online}=="1", RUN+="/usr/bin/sh -c '/usr/sbin/systemctl stop battery-power.target && /usr/sbin/systemctl start ac-power.target'"
```
and finally reload everything as-needed:
```
systemctl daemon-reload
udevadm control --reload-rules
udevadm trigger
```
Now, set up services as wanted, for example, you'll likely want to do this for `beesd`:
```
mkdir -p /etc/systemd/system/beesd\@.service.d/
```
In there, create `/etc/systemd/system/beesd\@.service.d/only-on-ac.conf` with content:
```
[Unit]
PartOf=ac-power.target

[Install]
WantedBy=ac-power.target
```
To activate the `Install` part, you need to re-enable the service after reloading `systemd`, for example:
```
systemctl daemon-reload
systemctl enable beesd@b8a34ebc-029a-4c77-ac2c-33290c18b461.service
```
Now, the service should be switched with AC state. You can check like this:
```
systemctl status ac-power.target
systemctl status battery-power.target
```
and of course by investigating the logs of affected services.

## Intel Low-Power-Mode Daemon
Install service:
```
yay -S intel-lpmd
```
Enable it:
```
systemctl enable --now intel_lpmd
```
Still need to determine whether that helps (may need to set `intel_lpmd_control AUTO` or write an actual XML config).

Check in:
```
systemctl status intel-lpmd
```
then copy over config to more matching config, e.g.:
```
cp /etc/intel_lpmd/intel_lpmd_config_F6_M189.xml /etc/intel_lpmd/intel_lpmd_config_F6_M189_T17.xml
```
and edit to your liking, for example:
```
<PerformanceDef>-1</PerformanceDef>
<BalancedDef>0</BalancedDef>
<PowersaverDef>1</PowersaverDef>
```
Note: Switching to powersave with this example config means only the E cores will be used. In case performance is not required, this can increase battery runtime noticeably.
