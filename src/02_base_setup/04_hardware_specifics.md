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
```
yay -S gst-plugin-libcamera pipewire-libcamera
```
Restart user-level `pipewire` service afterwards:
```
systemctl restart --user pipewire
```
In Firefox, set `media.webrtc.camera.allow-pipewire` in `about:config` to `True.
In Chromium, set `enable-webrtc-pipewire-camera` in `chrome://flags/`.
Both browsers need a restart for things to work.

Sadly, I am currently still getting segmentation faults of Pipewire and also when trying:
```
gst-launch-1.0 libcamerasrc ! video/x-raw,format=RGBA,width=1920,height=1080,framerate=30/1 ! videoconvert ! ximagesink
```
but for example `qcam` works. However, trying with another resolution:
```
gst-launch-1.0 libcamerasrc ! video/x-raw,format=RGBA,width=1920,height=1200,framerate=30/1 ! videoconvert ! ximagesink
```
works fine. It still breaks in Firefox etc. as likely they use a different resolution by default.

You will also want to set up `v4l2-relayd` as follows:
```
yay -S v4l2loopback-dkms v4l2-relayd
```
Now, load the module:
```
sudo modprobe v4l2loopback
```
and create a config for `v4l2-relayd` and start it. For that, first create `/etc/v4l2-relayd.d/frontcam.conf` with content:
```
# GStreamer source element name:
VIDEOSRC="libcamerasrc"
#SPLASHSRC="filesrc location=/.../splash.png ! pngdec ! imagefreeze num-buffers=4 ! videoscale ! videoconvert"

# Output format, width, height, and frame rate:
FORMAT=NV12
WIDTH=1920
HEIGHT=1200
FRAMERATE=30/1

# Virtual video device name:
CARD_LABEL="Dummy video device \(0x0000\)"

# Extra options to pass to v4l2-relayd:
EXTRA_OPTS=-d
```
Then, let it generate a systemd service froim that and start things:
```
systemctl daemon-reload
systemctl start v4l2-relayd
```
You can now check the status via:
```
systemctl status v4l2-relayd@frontcam.service
```
In principle, the camera should now also be accessible as a V4L2 device, but it still fails for me.

However, the camera is accessible via OBS on my system. Hence, the service above was not enabled permanenty. The following kind of works:
* Set up OBS, add a pipewire recording device, and select the camera, use resolution 1920x1200 and match it to full recording screen.
* Start virtual camera from OBS. Note that you must `systemctl stop v4l2-relayd@frontcam.service` first.

Sadly, Firefox crashes trying to access that, even though:
```
mpv av://v4l2:/dev/video32
```
works fine.

## Hybrid graphics: Initialization order
In case you are using hybrid graphics, it might happen that the initialization order of both GPUs is not guaranteed, which can lead to issues down the line (e.g. outputs enumerated and hence named differently between restarts, EDID patches not applied etc.). An order can be enforced for example by creating the file `/etc/modprobe.d/nvidia-before-intel.conf` with content:
```
softdep i915 pre: nvidia_drm nvidia_modeset nvidia_uvm nvidia
```
and running:
```
mkinitcpio -P
```
afterwards.

## Applying EDID patches
In some cases, you may need to patch the EDID of your monitor, e.g. to force support for different refresh rates with Wayland (which does not support custom modelines on its own, unlike X11).
Note that an alternative to the approach described below are custom modelines which can be created via `kscreen-doctor` in KDE/Plasma environments.

To create a custom EDID, you can for example use [CRU](https://www.monitortests.com/forum/Thread-Custom-Resolution-Utility-CRU) via Wine. You might also have success with [edid-generator](https://github.com/akatrevorjay/edid-generator).

To apply a custom EDID, you can place a patched EDID e.g. in `/lib/firmware/edid/Samsung-Syncmaster-patched.edid` and then edit `/etc/mkinitcpio.conf` to contain:
```
FILES=( /lib/firmware/edid/Samsung-Syncmaster-patched.edid )
```
Then, add:
```
drm.edid_firmware=HDMI-A-1:edid/Samsung-Syncmaster-patched.edid
```
to the kernel command line, and run:
```
mkinitcpio -P
```
afterwards and reboot the system.
