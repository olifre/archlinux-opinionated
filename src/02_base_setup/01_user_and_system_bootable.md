# Creating a user and making system self-bootable

## Activate `sshd`
```
systemctl enable --now sshd
```

## Create user
```
useradd -m olifre
passwd olifre
```

## Remote login
For more comfort (e.g. to set up a system with small screen or keyboard), you may now want to log in via `ssh` remotely. 

## Securing SSH
Configure `sshd`, i.e. copy over pubkey with `ssh-copy-id`, then set in `/etc/ssh/sshd_config`:
```
PasswordAuthentication no
```
and restart service:
```
systemctl restart sshd
```

## Install `yay`
Install `yay`, see [GitHub project](https://github.com/Jguer/yay) (will be needed for easier installation of bootloader / `shim-signed`):
```
pacman -Sy vi vim
```
Need to grant `sudo` permissions to regular user so `yay` can be used:
```
visudo
```
There, allow `wheel` users with password to use `sudo`. Then:
```
usermod -a -G wheel olifre
```

Then, as user (ensure to re-login to get group membership!):
```
sudo pacman -S --needed git base-devel
mkdir AUR
cd AUR
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
yay -Y --devel --save
```
Configure for speed, create `/etc/makepkg.conf.d/multicore.conf` with content:
```
NPROC=8
MAKEFLAGS="-j8"
```

## Install the bootloader
We can finally install the boot loader. We will be using Secure Boot, with MOK (Machine-Owner Keys), so do as user:
```
yay refind sbsigntools
yay shim-signed
```
Then, as `root`, run:
```
refind-install --shim /usr/share/shim-signed/shimx64.efi --localkeys
```
Configure `refind` by editing `/efi/EFI/refind/refind.conf`. Notably, comment out all those dummy `menuentry` a the bottom, and make sure to add those settings:
```
timeout 15
use_nvram false
banner dell_logo.bmp
use_graphics_for osx,linux,windows
showtools shell, bootorder, gdisk, memtest, mok_tool, apple_recovery, windows_recovery, about, hidden_tags, reboot, exit, firmware, fwupdate
fold_linux_kernels false
extra_kernel_version_strings "linux-hardened,linux-rt-lts,linux-zen,linux-lts,linux-rt,linux"
write_systemd_vars true
```
For ease of maintainability, you may want to add these below to the existing descriptions. Note the `write_systemd_vars` is critical for GPT automounting (i.e. this will cause the partitions to be detected and mounted automatically from the same disk which holds the ESP), `extra_kernel_version_strings` is important for the Arch kernel naming scheme, and `fold_linux_kernels` is helpful in case you also want to install the LTS kernel later on.

Copy over an icon from UEFI BGRT for themeing and a kernel icon:
```
cp /sys/firmware/acpi/bgrt/image /efi/EFI/refind/dell_logo.bmp
cp /efi/EFI/refind/icons/os_arch.png /boot/vmlinuz-linux.png
cp /efi/EFI/refind/icons/os_arch.png /boot/vmlinuz-linux-lts.png
```
Now, follow [the ArchWiki on how to set up kernel signing](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Signing_the_kernel_with_a_mkinitcpio_post_hook), and put the following in the `hook` file:
```
keypairs=(/etc/refind.d/keys/refind_local.key /etc/refind.d/keys/refind_local.crt)
```
Do not forget to make the hook executable! Then, as user:
```
yay linux
```
This will reinstall the kernel and sign it. 

Also, add a `refind` update hook, create `/etc/pacman.d/hooks/refind.hook` with content:
```
[Trigger]
Operation=Upgrade
Type=Package
Target=refind

[Action]
Description = Updating rEFInd on ESP
When=PostTransaction
Exec=/usr/bin/refind-install --shim /usr/share/shim-signed/shimx64.efi --localkeys
```

Now, a reboot should show a Secure Boot warning and MokManager should pop up. In there, enroll the MOK from:
```
esp/EFI/refind/keys/refind_local.cer
```

If this works as expected, you will have a booting system! You can unenroll the key of your Ventoy medium (if used) at this point.
