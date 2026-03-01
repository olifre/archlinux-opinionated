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

ACTION=="add", SUBSYSTEM=="usb", TEST=="power/control", ATTR{power/control}="auto"
LABEL="power_usb_rules_end"
```
Note that the `blacklist` can be used in case you encounter a device which has problems with that. 
You may want to run `mkinitcpio -P` afterwards. 
