# mac-artix
> Artix (With Runit) dual boot installation guide for Macbook

<p align="center">
  <img src="demo.png" width="1000">
</p>

- :ballot_box_with_check: display
- :ballot_box_with_check: audio
- :ballot_box_with_check: internet connection, wifi
- :ballot_box_with_check: keyboard (work perfectly as normal keyboard)
- :ballot_box_with_check: trackpad & external mouse (with natural scroll and more)
- :ballot_box_with_check: screen backlight
- :ballot_box_with_check: keyboard backlight
- :ballot_box_with_check: fan
- :ballot_box_with_check: battery


# Contents

- [Before you start](#before-you-start)
- [Install Artix dual boot](#install-artix-dual-boot)
  - [Make space for Artix](#make-space-for-artix)
  - [Make installer USB](#make-installer-usb)
  - [Boot it up](#boot-it-up)
  - [Login](#login)
  - [Partition](#partition)
  - [Format and Mount](#format-and-mount)
  - [Connect Wifi](#connect-wifi)
  - [Install Base System](#install-base-system)
  - [Install Kernel](#install-kernel)
  - [Generate fstab](#generate-fstab)
  - [Configure the Base System](#configure-the-base-system)
  - [Configure Network](#configure-network)
  - [Install the Bootloader](#install-the-bootloader)
  - [Reboot the System](#reboot-the-system)
  - [Make Artix Dual Bootable](#make-artix-dual-bootable)
- [Install to make artix usable](#install-to-make-artix-usable)
  - [Set tty default font](#set-tty-default-font)
  - [Install drivers](#install-drivers)
  - [Install require packages](#install-require-packages)
  - [Install Window Manager](install-window-manager)


# Before you start

It's seem [Macbook with T2 Security](https://support.apple.com/en-us/HT208862) isn't support linux very well, view this [discussions](https://discussions.apple.com/thread/251087440?answerId=252062188022#252062188022)

# Install Artix dual boot

## Make space for Artix

Use Disk Utility Partition feature to add new Partition for Artix, follow [this guide](https://wiki.archlinux.org/index.php/Mac#Arch_Linux_with_OS_X_or_other_operating_systems)

Or if you already know how to use Disk Utility, then create a partition with FAT32 format.

## Make installer USB

Download Artix base ISO [here](https://iso.artixlinux.org/isos.php)

Find usb by using `lsblk` or `diskutil list`, etc..., then:

```sh
# Assume usb disk is /dev/diskX
umount /dev/diskX
dd if=path/to/arch.iso of=/dev/diskX bs==1m
```

## Boot it up

Hold <kbd>alt/option</kbd> when system bootup, then choose boot from USB.

:warning: If you are using Retina Macbook, tty font will be very small. To get larger font, [connect to wifi](#connect-wifi) and run these commands:

```sh
sudo pacman -Sy terminus-font
setfont /usr/share/kbd/consolefonts/ter-132b.psf.gz
```

Or you can use some font already in `/usr/share/kbd/consolefonts`

## Login

After boot up, choose `keytable`, `lang` or leave it default if you not sure what are they, then choose:

`[From CD/DVD/ISO: artix.x86_64]`

Then login with:

```sh
Username: artix
Password: artix
```

## Partition

View all your partitions to choose correct one with:

```sh
lsblk
```

Then open cfdisk to partition own disk

```sh
# Assume my disk is /dev/sda
cfdisk /dev/sda
```

Then create these new partitions:

|Size    |Type              |Description|
|---     |---               |---        |
|128MB   |Apple HFS+        |This is required in order to make artix dual boot with OSX|
|256MB   |Linux filesystem  |Artix file system|
|xMB     |Linux Swap        |If you have space, try to make it double size of your ram size|
|xGB     |Linux filesystem  |This is our home|

## Format and Mount

Assuming you have this after partitioning

|Device             |Size            |Type|
|---                |---             |---|
|/dev/sda3          |128MB           |Apple HFS+|
|/dev/sda4          |256MB           |Linux filesystem|
|/dev/sda5          |16GB            |Linux Swap|
|/dev/sda6          |64GB            |Linux filesystem|

Now let format it all:

```sh
mkfs.ext4 /dev/sda4
mkfs.ext4 /dev/sda6
mkswap /dev/sda5
```

Then mount & turn on swap:

```sh
mount /dev/sda6 /mnt
mkdir /mnt/boot
mount /dev/sda4 /mnt/boot
swapon /dev/sda5
```

## Connect Wifi

Connect with [connman](https://wiki.archlinux.org/index.php/ConnMan)

Example:

```sh
connmanctl              # Open connman
enable wifi             # Enable wifi
scan wifi               # Scan wifi
agent on                # Enable wireless agent
services                # List all scanned wifi
connect wifi_XXX        # Connect with wifi_XXX goes after your wifi name
```

Then check connection with:

```sh
ping -c 3 google.com
```

## Install Base System

I'm using runit so:

```sh
basestrap /mnt base base-devel runit elogind-runit
```

## Install Kernel

You can choose `linux` or `linux-lts`. I'v tried `linux` kernel on my [MJLQ2-MBP](https://support.apple.com/kb/sp719?locale=en_VN) but it cause udev stuck and we have to fix by edit grub default like:

```sh
GRUB_CMDLINE_LINUX_DEFAULT="nomodeset quiet rootflags=data=writeback"
```

Although that fix udev get stuck but then you will can't control backlight of your MBP

So, i'v install `linux-lts` and everything work out of the box:

```sh
basestrap /mnt linux-lts linux-firmware
```

## Generate fstab

Run this command:

```sh
fstabgen -U /mnt >> /mnt/etc/fstab
```

:warning: If you are using SSD Drive, Open fstab config file:

```sh
vim /mnt/etc/fstab
```

Remove all `discard` in all lines & edit everything to look like:

/dev/sda4   /boot   ext2   defaults,relatime,stripe=4        0 2
/dev/sda6   /       ext4   defaults,noatime,data=writeback   0 1

## Configure the Base System

Let chroot into own new Artix system:

```sh
artools-chroot /mnt
```

Set system clock:

```sh
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

Localization:

```sh
# Uncomment `en_US.UTF-8 UTF-8` (Or what ever locale you want) line in `/etc/locale.gen` file, then run:
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

Add user:

```sh
useradd -m -G wheel your_username
passwd your_username   (Create your password)

# Also change your sudo password with:
passwd
```

Add sudo rights for our user by open `/etc/sudoers` file and uncomment this line:

```sh
%wheel ALL=(ALL) ALL
```

Create an initial ramdisk environment:

```sh
mkinitcpio -p linux-lts
# or linux if you using linux kernel
```

## Configure Network

Set hostname:

```sh
echo artix > /etc/hostname   (Change artix with your preper hostname)
```

Install there packages:

```sh
pacman -S connman-runit dhcpcd wpa_supplicant
```

:warning: If you only have wireless connection, be sure to install `wpa_supplicant` and make sure it install successfully

Enable connmand service:

```sh
ln -s /etc/runit/sv/connmand /etc/runit/runsvdir/default
```

## Install the Bootloader

We will boot using OSX native EFI boot loader, so install this:

```sh
pacman -S grub-efi-x86_64
```

Change `/etc/default/grub` to look like:

```sh
GRUB_CMDLINE_LINUX_DEFAULT="quiet rootflags=data=writeback"
```

⚠ If you using `linux` kernel instead of `linux-lts` kernel, you may try this:

```sh
GRUB_CMDLINE_LINUX_DEFAULT="nomodeset quiet rootflags=data=writeback"
```

Then create boot.efi with GRUB:

```sh
# Create empty "boot/grub/grub.cfg" file if it not exist
grub-mkconfig -o boot/grub/grub.cfg
grub-mkstandalone -o boot.efi -d usr/lib/grub/x86_64-efi -O x86_64-efi --compress xz boot/grub/grub.cfg
```

:heavy_exclamation_mark: Important: Copy boot.efi to your usb or upload it somewhere, we'll need this to dual boot with OSX.

To copy it to usb, use:

```
mkdir /mnt/myusb && mount /dev/sdb /mnt/myusb
cp boot.efi /mnt/myusb/
```

Or upload it to [file.io](https://www.file.io/):

```
curl -F "file=@boot.efi" https://file.io
```

## Reboot the System

If everything ok, now you can exit chroot and reboot (Back to OSX word)

```sh
exit             <- (exit chroot environment)
umount -R /mnt
reboot
```

## Make Artix Dual Bootable

When OSX loaded. Using Disk Utility to format 128MB Apple HFS+ we have created & formatted before with `Journaled` format.

Then create this file structure inside:

```
|___mach_kernel   
|___System   
    |___Library   
        |___CoreServices   
            |___SystemVersion.plist   
            |___boot.efi              <- (is the boot.efi file we'v copied or uploaded in the previous step)  
```

Add below content to `SystemVersion.plist`:

```xml
<xml version="1.0" encoding="utf-8"?>
<plist version="1.0">
<dict>
    <key>ProductBuildVersion</key>
    <string></string>
    <key>ProductName</key>
    <string>Linux</string>
    <key>ProductVersion</key>
    <string>Artix Linux</string>
</dict>
</plist>
```

Then reboot and hold <kbd>alt/option</kbd> and enjoy Artix 😺

# Install to make artix usable

## Set tty default font

:warning: Font in retina screen is very small, so this maybe the first step you should do . If you can see the font clearly, you don't have to do this step.

Install terminus-font (or whatever font you prefer, make sure it have large size): 

```sh
sudo pacman -S terminus-font
```

create file `/etc/vconsole.conf` with content:

```
FONT=ter-132b
```

Install these default fonts (to make browser look suckless):

```
yay -S ttf-mac-fonts ttf-ms-fonts ttf-opensans
```

## Install drivers

```
sudo pacman -S xf86-video-intel xf86-input-libinput mesa
```

## Install require packages

Using pacman to install all these packages:   

|Package       | Description |
|---           |---          |
|xorg-server   | graphical server |
|xorg-xinit    | starts graphical server |
|xorg-xrandr   | resize & rotate utility for X |
|xorg-xsetroot | utility to set your root window background to a given pattern or color |
|xorg-xev      | indentifying keycodes |
|picom         | lightweight compositor for X11 |
|xwallpaper    | set wallpaper |
|arandr        | UI for screen adjustment |

## Install Window Manager

I build my own [dwm](https://github.com/kyoz/dwm), [dmenu](https://github.com/kyoz/dmenu), [st](https://github.com/kyoz/st) from [suckless](https://suckless.org/)

But you can install any window manager you like



## Bluetooth
pacman -S bluez bluez-runit bluez-utils
ln -s /etc/runit/sv/bluetoothd /run/runit/service
sv start bluetoothd
sv restart bluetoothd

For bluetooth audio device, install these
```sh
sudo pacman -S pulseaudio-alsa pulseaudio-bluetooth
```

You may need install blueman for easily manage your bluetooth devices
