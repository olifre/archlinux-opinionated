# Hardware specifics

## IPU 7 webcam
This follows along the [Guide on the ArchWiki](https://wiki.archlinux.org/title/Dell_XPS_13_(9350)_2024#Camera) for a specific laptop model. 

Before starting, you'll want to install `libcamera-tools` to check whether the situation has improved in the meantime:
```
yay -S libcamera-tools
```
You can use:
```
cam -l
qcam
```
for testing things.

If this does not show something useful (see the [Guide on the ArchWiki](https://wiki.archlinux.org/title/Dell_XPS_13_(9350)_2024#Camera), install DKMS and kernel headers so DKMS can be used:
```
yay -S dkms linux-lts-headers linux-headers
```
Then, install Intel Vision Drivers (DKMS) from AUR:
```
yay -S intel-vision-drivers-dkms-git
```
Finally, create `/etc/modules-load.d/intel_cvs.conf` with content:
```
intel_cvs
```
and run, just to be safe:
```
mkinitcpio -P
```
and reboot. The commands from above should work now, you should see a camera picture with non-ideal quality.

Now, install the tooling necessary to use the camera in actual applications.

FIXME, this is still work in progress!
