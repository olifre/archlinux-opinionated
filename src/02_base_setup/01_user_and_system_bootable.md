# Creating a user and making system self-bootable

## Activate `sshd`
```
systemctl enable --now sshd
```

## Create user
```
useradd -m olifre
passwd olifre
loginctl enable-linger olifre
```
Note: The `linger` feature will allow some user-level services to start even before login, e.g. Syncthing, userspace Wireguard etc.

## Remote login
For more comfort (e.g. to set up a system with small screen or keyboard), you may now want to log in via `ssh` remotely. 

## Securing SSH
Configure `sshd`, i.e. copy over pubkey with `ssh-copy-id`, then set in `/etc/ssh/sshd_config.d/90-userconfig.conf` (file has to be newly created):
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
NPROC="$(nproc)"
MAKEFLAGS="-j$(nproc)"
```

## Adapt the `/etc/fstab`
In preparation for `btrbk` and more, you should adapt the `/etc/fstab`. It should look like this:
```
# <file system> <dir> <type> <options> <dump> <pass>
# /dev/mapper/root
UUID=8fae96ce-42b0-4933-88ac-f4cdb41155ad       /               btrfs           rw,noatime,commit=60,compress-force=zstd:6,ssd,space_cache=v2,subvol=/rootfs      0 0

# /dev/mapper/root
UUID=8fae96ce-42b0-4933-88ac-f4cdb41155ad       /home           btrfs           rw,noatime,commit=60,compress-force=zstd:6,ssd,space_cache=v2,subvol=/home        0 0

# /dev/mapper/root pool directory
UUID=8fae96ce-42b0-4933-88ac-f4cdb41155ad       /mnt/btrfs_pool btrfs           rw,noatime,commit=60,compress-force=zstd:6,ssd,space_cache=v2,subvolid=5,noauto   0 0

# /dev/nvme0n1p1
UUID=1542-2E81          /efi            vfat            noauto,x-systemd.automount,x-systemd.idle-timeout=1min,rw,relatime,fmask=0027,dmask=0027,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro   0 2

# /dev/nvme0n1p2
UUID=8fae96ce-42b0-4933-88ac-f4cdb41155ad       /boot           ext4            noauto,commit=60,x-systemd.automount,x-systemd.idle-timeout=1min,rw,relatime      0 2
```
The important things we added here are the `/mnt/btrfs_pool` mountpoint and the `automount` settings for `/efi` amnd `/boot` such that they should only be mounted when actually accessed. You may want to regenerate the `initrd` at this point:
```
mkinitcpio -P
```
and you should for sure generate the `/mnt/btrfs_pool` mountpoint:
```
mkdir /mnt/btrfs_pool
```
and of course make sure the UUIDs match your system (use `blkid` to check)!

## Set up `discard` for crypto devices, and increase performance for SSDs
Be sure you are aware of the security implications! We do this to increase the SSD lifetime.
For the performance trick, see the [ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Disable_workqueue_for_increased_solid_state_drive_(SSD)_performance) for more details.
```
cryptsetup --allow-discards --perf-no_read_workqueue --perf-no_write_workqueue --persistent refresh root
```

## Install the bootloader `refind`
We can finally install the boot loader `refind`. We will be using Secure Boot, with MOK (Machine-Owner Keys), so do as user:
```
yay -S refind sbsigntools shim-signed
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
Now, follow the [ArchWiki on how to set up kernel signing](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Signing_the_kernel_with_a_mkinitcpio_post_hook), and put the following in the `hook` file:
```
keypairs=(/etc/refind.d/keys/refind_local.key /etc/refind.d/keys/refind_local.crt)
```
so the total file should look like:
```
#!/usr/bin/env bash

kernel="$1"
[[ -n "$kernel" ]] || exit 0

# use already installed kernel if it exists
[[ ! -f "$KERNELDESTINATION" ]] || kernel="$KERNELDESTINATION"

keypairs=(/etc/refind.d/keys/refind_local.key /etc/refind.d/keys/refind_local.crt)

for (( i=0; i<${#keypairs[@]}; i+=2 )); do
    key="${keypairs[$i]}" cert="${keypairs[(( i + 1 ))]}"
    if ! sbverify --cert "$cert" "$kernel" &>/dev/null; then
        sbsign --key "$key" --cert "$cert" --output "$kernel" "$kernel"
    fi
done
```
The hook file should be saved as `/etc/initcpio/post/10-kernel-sbsign` so ordering against other hooks is possible. Do not forget to make the hook executable!

We will also add another hook as a small "hack" so the kernel and `initrd` are also present on the ESP in case something goes from with the `XBOOTLDR` partition (or e.g. the `refind` filesystem drivers fail). For this, create a hook file `/etc/initcpio/post/95-copy-to-esp` with content:
```
#!/usr/bin/env bash

ESP_DIR=/efi/EFI/arch
if [[ ! -d "$ESP_DIR" ]]; then
    echo "ESP directory ${ESP_DIR} does not exist, not copying kernel there!"
    exit 0
fi

kernel="$1"
[[ -n "$kernel" ]] || exit 0

initrd="$2"
[[ -n "$initrd" ]] || exit 0

BOOTDIR=$(dirname "$KERNELDESTINATION")
if [[ ! -d "$BOOTDIR" ]]; then
    echo "Boot directory ${BOOTDIR} which should hold kernel and initrd does not exist, not copying..."
    exit 0
fi

# Copy over kernel and initrd:
cp -av "$kernel" "$ESP_DIR"/
cp -av "$initrd" "$ESP_DIR"/

# Optional UKI would reside on ESP already, no copying.
if [ $# -eq 3 ]; then
    uki=$3
    #cp -av "$uki" "$ESP_DIR"/
fi

# Copy over refind config if it exists:
if [[ -f "$BOOTDIR"/refind_linux.conf ]]; then
    cp -av "$BOOTDIR"/refind_linux.conf "$ESP_DIR"/
fi
```
Do not forget to make it executable.

Note you must also create `/efi/EFI/arch/` (which will be used for UKIs later) and `/efi/EFI/systemd/` (which will be used for `fwupd` later, note that it prefers `systemd` over `arch`):
```
mkdir -p /efi/EFI/arch/
mkdir -p /efi/EFI/systemd/
```

Then, as user:
```
yay -S linux linux-lts
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
To ensure `shim-signed` stays up to date on the ESP, also create a hook for that by creating `/etc/pacman.d/hooks/shim-signed.hook` with content:
```
[Trigger]
Operation=Upgrade
Type=Package
Target=shim-signed

[Action]
Description = Updating Shim on ESP
When=PostTransaction
Exec=/bin/sh -c "/usr/bin/cp /usr/share/shim-signed/shimx64.efi /efi/EFI/arch/shimx64.efi && /usr/bin/cp /usr/share/shim-signed/shimx64.efi /efi/EFI/systemd/shimx64.efi && /usr/bin/cp /usr/share/shim-signed/shimx64.efi /efi/EFI/refind/shimx64.efi"
```

Now, a reboot should show a Secure Boot warning and MokManager should pop up. In there, enroll the MOK from:
```
esp/EFI/refind/keys/refind_local.cer
```

If this works as expected, you will have a booting system! You can unenroll the key of your Ventoy medium (if used) at this point.


## Fallback solution: UKIs

You may want to enable the build of [Unified kernel images](https://wiki.archlinux.org/title/Unified_kernel_image), which can then easily be booted by simple loaders such as `systemd-boot`.
To do so, create the directory for kernel commandline snippets:
```
mkdir /etc/cmdline.d/
```
and then create files as follows:
```
echo "rw" > /etc/cmdline.d/root.conf
echo "zswap.enabled=0" > /etc/cmdline.d/disable-zswap.conf
echo "rd.luks.name=bdf5159d-ad5d-4d1d-bfac-ce833c92048d=root rd.luks.options=bdf5159d-ad5d-4d1d-bfac-ce833c92048d=tries=10,timeout=0 rootflags=x-systemd.device-timeout=0" > /etc/cmdline.d/luks.conf
echo "quiet splash" > /etc/cmdline.d/plymouth.conf
```
Note that the UUID is used here explicitly (see also [Add `refind_linux.conf` for boot configuration](../01_pre_boot/02_bootstrapping_arch_initial_config.md#refindlinuxconf) on the UUID) as especially for UKIs, `sd-encrypt` seems not to allow for any retries at all in case of mistyped passwords and settings may be ignored unless passed in with UUID explicitly.

Now, edit `/etc/mkinitcpio.d/linux.preset` and `/etc/mkinitcpio.d/linux-lts.preset`, in both, uncomment th lines `default_uki` and `default_options`.

Then, create a hook to sign those UKIs with out MOK. Create the file `/etc/initcpio/post/11-uki-sbsign` with content:
```
#!/usr/bin/env bash

uki="$3"
[[ -n "$uki" ]] || exit 0

key=/etc/refind.d/keys/refind_local.key
cert=/etc/refind.d/keys/refind_local.crt

if ! sbverify --cert "$cert" "$uki" &>/dev/null; then
    sbsign "$uki" --key "$key" --cert "$cert" --output "$uki"
fi
```
and make sure to mark this file executable!


Finally, create the directory for the UKIs and call `mkinitcpio`:
```
mkdir -p /efi/EFI/Linux/
mkinitcpio -P
```
You should now find UKIs in `/efi/EFI/Linux/`.

In case you want to add boot entries for these directly to your firmware, be aware that they need to be loaded via `shim` (since a MOK was used for signing). The following two commands can be used:
```
efibootmgr --disk /dev/nvme0n1 --part 1 --create --label "ArchLinux linux" --loader '\EFI\refind\shimx64.efi' --unicode '\EFI\Linux\arch-linux.efi '
efibootmgr --disk /dev/nvme0n1 --part 1 --create --label "ArchLinux linux-lts" --loader '\EFI\refind\shimx64.efi' --unicode '\EFI\Linux\arch-linux-lts.efi '
```
See more details about the "space" at the end of the loader name in the next section, and about potential hacks in case this does not work with your firmware due to bugs.

In case you use this, please check and adapt your boot order to your liking afterwards!


## Fallback solution: `systemd-boot`
In case you have set up UKI building, installing `systemd-boot` is rather easy. Before starting, please note that it will replace the fallback boot loader in addition to installing itself, which might interfere with how Windows expects things to be in case you dual-boot. To go ahead, execute:
```
bootctl install
```
and afterwards inspect `efibootmgr` and adapt your boot order as you like (by default, `systemd-boot` will register itself as the first loader).
In case you do not want to use UKIs, this is also possible, but not covered here, check [this ArchWiki entry](https://wiki.archlinux.org/title/Systemd-boot) for details on how to achieve that.

You can then check with:
```
bootctl status
```
whether the setup went well, and:
```
bootctl list
```
should list the UKIs.

Now, to set up Secure Boot with the MOK created earlier, create the file `/etc/pacman.d/hooks/80-systemd-secureboot.hook` with content:
```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/systemd/boot/efi/systemd-boot*.efi

[Action]
Description = Signing systemd-boot EFI binary for Secure Boot
When = PostTransaction
Exec = /bin/sh -c 'while read -r f; do /usr/lib/systemd/systemd-sbsign sign --private-key /etc/refind.d/keys/refind_local.key --certificate /etc/refind.d/keys/refind_local.crt --output "${f}.signed" "$f"; done;'
Depends = sh
NeedsTargets
```
Furthermore, to automate updates, create `/etc/pacman.d/hooks/95-systemd-boot.hook` with content:
```
[Trigger]
Type = Package
Operation = Upgrade
Target = systemd

[Action]
Description = Gracefully upgrading systemd-boot...
When = PostTransaction
Exec = /usr/bin/systemctl restart systemd-boot-update.service
```
You need to perform:
```
pacman -S systemd
```
once afterwards to trigger the hooks, and finally:
```
bootctl install
```
once more, as the update would otherwise skip updating the same version (which we require as the old one is not signed).
Remember to check your boot order afterwards!

In the end, this setup is not bootable just yet, as with a MOK, `shim` needs to be used. For that, we can re-use the `shim` deployed with `refind` and already updated as described above. To do so, add your own boot option as follows:
```
efibootmgr --disk /dev/nvme0n1 --part 1 --create --label "Linux Secure Boot" --loader '\EFI\refind\shimx64.efi' --unicode '\EFI\systemd\systemd-bootx64.efi '
```
Note that the "space" at the end is on purpose, as some firmwares do not seem to safely terminate string payloads without an explicit space, as described in [this GitHub issue](https://github.com/systemd/systemd/issues/27234#issuecomment-2902199758). If even that fails with your firmware, you'd need a separate script to keep `shim` updated, copy it to the `systemd` folder, and rename `systemd-bootx64.efi` to `grubx64.efi` via another hook. Inspiration can be taken from [this ArchLinux forum thread](https://bbs.archlinux.org/viewtopic.php?id=293365).
Again, make sure the boot order matches what you want after this step!

Finally, you can configure `systemd-boot` to your liking. Edit `/efi/loader/loader.conf` to contain:
```
timeout 3
console-mode keep
editor yes
```
Note that the editor will not work in Secure Boot mode.

Note: In case you switched to `systemd-boot` later on, you might want to purge existing `fwupd` installs in `/efi/EFI/arch/` as `fwupd` will reinstall itself to `/efi/EFI/systemd` the next time you trigger an update.

