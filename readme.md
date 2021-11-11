# How to install Arch Linux, my way

Installing Arch Linux is all about picking every aspect of the system to one's exact preference.
I have played with "easier" distros like ArcoLinux recently, but I find that they end up installing some things I don't need, and it is more work for me to remove everything I don't need than running an install from scratch.

That being said, there is an appeal to having a consistent method to setup a distro exactly as intended, as fast as possible.

Here, I will try to create a step-by-step guide for myself that I can follow for future installations to save time on them.

# General goals

* System: Use the most modern approach that's reliable
  * UEFI
  * systemd-boot
  * systemd-mount
  * fwupd
  * btrfs
  * encryption
  * wayland
* Desktop environment: gnome for casual use + some wayland tiling window manager for focused work (sway for now)
  * gnome
  * sway
* Applications:
  * zsh
  * rust programming toolchain
  * basic utilities like a couple of web browsers, a graphical text editor
* Theming:
  * fonts
  * colorschemes/customizations


# Download the iso and create a bootable medium:

* [Download](https://archlinux.org/download/) the latest iso.

* With the USB drive inserted, but not mounted, and its name (say `/dev/sdx`) known thanks to `lsblk`.
```
$ dd bs=4M if=path/to/archlinux-version-x86_64.iso of=/dev/sdx conv=fsync oflag=direct status=progress
```

# Connect to wifi

* Boot into the install medium

* Use [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl) to connect to the wifi
```
$ systemctl start iwd                  # Start the systemd service
$ iwctl                                # Run iwctl
[iwd]# device list                     # Get the device (for instance, wlan0)
[iwd]# station <device> scan           # Scan for networks
[iwd]# station <device> get-networks   # List ssid for available networks
[iwd]# station <device> connect <ssid> # Connect to desired ssid
ctrl + D
ping 8.8.8.8
```

# Make sure the system clock is up-to-date

```
# timedatectl set-ntp true
```

# Partition the disk, setup btrfs and encryption

* These steps come from [here](https://austinmorlan.com/posts/arch_linux_install/)
* Identify the disk to partition with `lsblk` (say /dev/sda)
* Create boot partition and main partition to encrypt with `gdisk`
```
gdisk /dev/sda
o (Create a new empty GUID partition table (GPT))
Proceed? Y
n (Add a new partition)
Partition number 1
First sector (default)
Last sector +512M
Hex code EF00
n (Add a new partition)
Partition number 2
First sector (default)
Last sector (press Enter to use remaining disk)
Hex code 8300
w
Y
```
* Set up the encryption container:
```
cryptsetup luksFormat /dev/sda2
Are you sure? YES
Enter passphrase (twice)
cryptsetup open /dev/sda2 luks
```
* Format the boot partition with FAT32 and the root partition with btrfs
```
mkfs.vfat -F32 /dev/sda2
mkfs.btrfs /dev/mapper/luks
```
* Mount btrfs volume with appropriate options (quoting Austin Morlan)
> Since I’d like to keep separate snapshots of home in case a root snapshot needs to be restored, I create a subvolume for both (see here for details). I also create one for /var so that its noisy state isn’t included in snapshots.

> `noatime` and `nodiratime` are used to prevent a write every time a file or directory is accessed (not great for a COW filesystem like btrfs).

> `lzo` is used for compression because it’s faster at the expense of disk space (of which I have plenty).

> I don’t use `discard` because it has a performance cost: I just issue manual trim commands with fstrim.
	
```
mount /dev/mapper/luks /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
umount /mnt
mount -o subvol=@,ssd,compress=lzo,noatime,nodiratime /dev/mapper/luks /mnt
mkdir /mnt/{boot,home,var}
mount -o subvol=@home,ssd,compress=lzo,noatime,nodiratime /dev/mapper/luks /mnt/home
mount -o subvol=@var,ssd,compress=lzo,noatime,nodiratime /dev/mapper/luks /mnt/var
mount /dev/sda1 /mnt/boot
```

# Install and configure the system

* Install some necessities (note: replace `intel-ucode` with `amd-ucode` on `amd`)
```
pacstrap /mnt base base-devel linux linux-firmware btrfs-progs neovim vi sudo zsh zsh-autosuggestions zsh-completions zsh-syntax-highlighting zsh-theme-powerlevel10k iwd fd ripgrep bat intel-ucode
```
* Generate the partition list in the fstab file
```
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/fstab # Check it looks good (boot partition and all btrfs volumes present)
```
* Remove files so systemd-firstboot can generate them
```
rm /mnt/etc/{machine-id,shadow}
cp /etc/pacman.d/mirrorlist /mnt/etc/pacman.d/mirrorlist
```
* chroot and set up some things, including [systemd-boot](https://wiki.archlinux.org/title/systemd-boot)
```
arch-chroot /mnt /bin/zsh
chsh -s zsh
nvim /usr/lib/systemd/system/systemd-firstboot.service # edit (see wiki)
systemctl enable systemd-firstboot
nvim /etc/locale.gen (uncomment en_US.UTF-8 and en_GB.UTF-8)
locale-gen
echo LANG=en_GB.UTF-8 > /etc/locale.conf
passwd
```
Edit /etc/hosts:

```
127.0.0.1   localhost
::1     localhost
127.0.1.1   some_hostname.localdomain   some_hostname
```
* Modify /etc/mkinitcpio.conf to add encrypt and btrfs:
```
HOOKS="base udev autodetect modconf block encrypt btrfs filesystems keyboard fsck"
```
* Generate initcpio
```
mkinitcpio -p linux
```
* Install systemd-boot
```
bootctl --path=/boot install
```
* Get UUID of cryptoLUKS partition with `blkid`

* Create `/boot/loader/entries/arch.conf`:
```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=UUID_FROM_ABOVE:luks:allow-discards root=/dev/mapper/luks rootflags=subvol=@ rd.luks.options=discard rw mem_sleep_default=deep
```
* Edit /boot/loader/loader.conf:
```
default arch
```
* Reboot:

```
exit
umount -R /mnt
reboot
```

# First boot

As prompted systemd-firstboot, set the locale (`uk` or `us`, typically), the hostname and the timezone (`Europe/London`)

* Connect to the internet again with [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl) to connect to the wifi
```
$ systemctl start iwd                  # Start the systemd service
$ iwctl                                # Run iwctl
[iwd]# device list                     # Get the device (for instance, wlan0)
[iwd]# station <device> scan           # Scan for networks
[iwd]# station <device> get-networks   # List ssid for available networks
[iwd]# station <device> connect <ssid> # Connect to desired ssid
ctrl + D
ping 8.8.8.8
```

* [Create a user](https://wiki.archlinux.org/title/Users_and_groups)
```
useradd pierrec -m -G wheel,sudo -s /bin/zsh
passwd pierrec
```

* Give it super powers
```
visudo
#uncomment line: "%wheel ALL=(ALL) ALL"
#uncomment line: "%sudo ALL=(ALL) ALL"
```

* Setup automatic mirrors management using [reflector](https://wiki.archlinux.org/title/Reflector)
```
sudo pacman -S reflector
sudo pacman -S start reflector.timer
sudo pacman -S enable reflector.timer
```

* Create pacman hook to start reflector.service on pacman-mirrorlist upgrade:
```
/etc/pacman.d/hooks/mirrorupgrade.hook
---

[Trigger]
Operation = Upgrade
Type = Package
Target = pacman-mirrorlist

[Action]
Description = updating pacman-mirrorlist with reflector and removing pacnew...
When = PostTransaction
Depends = reflector
Exec = /bin/sh -c 'systemctl start reflector.service; [ -f /etc/pacman.d/mirrorlist.pacnew ] && rm /etc/pacman.d/mirrorlist.pacnew'
```

* Install gnome
```
sudo pacman -S gnome gnome-extra
sudo systemctl enable gdm.service
```
* Install and setup NetworkManager
```
sudo pacman -S networkmanager
```
* Use iwd as the backend by creating this file:
```
/etc/NetworkManager/conf.d/wifi_backend.conf
---
[device]
wifi.backend=iwd
```
* Enable networking
```
sudo systemctl start NetworkManager
sudo systemctl enable NetworkManager
```

* Install desired applications
```
sudo pacman -S firefox alacritty discord exa ninja cmake git rustup transmission-gtk
```

* Setup ssh keys
```
ssh-keygen
```
* Log-in to the github web interface and add the public key (from `~/.ssh/id_rsa.pub`

* Setup rust environment by following [upstream instructions](https://www.rust-lang.org/learn/get-started)
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

* Install [yay](https://github.com/Jguer/yay), an aur hepler that gets out of your way
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

* Install aur packages
```
yay -S google-chrome
```

* Pimp it!
```
sudo pacman -S arc-gtk-theme arc-icon-theme
```
* Open `gnome-tweaks` and set
  * Appearance >> Applications: Arc Dark
  * Appearance >> Icons: Arc

* Install [hack-nerdfont](https://github.com/ryanoasis/nerd-fonts/tree/master/patched-fonts/Hack#linux)
  * Download the latest complete version of Hack.
```
cd ~/Downloads
sudo pacman -S wget
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/Regular/complete/Hack%20Regular%20Nerd%20Font%20Complete.ttf
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/Regular/complete/Hack%20Regular%20Nerd%20Font%20Complete%20Mono.ttf
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/Italic/complete/Hack%20Italic%20Nerd%20Font%20Complete.ttf
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/Italic/complete/Hack%20Italic%20Nerd%20Font%20Complete%20Mono.ttf
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/Bold/complete/Hack%20Bold%20Nerd%20Font%20Complete.ttf
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/Bold/complete/Hack%20Bold%20Nerd%20Font%20Complete%20Mono.ttf
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/BoldItalic/complete/Hack%20Bold%20Italic%20Nerd%20Font%20Complete.ttf
wget https://github.com/ryanoasis/nerd-fonts/raw/master/patched-fonts/Hack/BoldItalic/complete/Hack%20Bold%20Italic%20Nerd%20Font%20Complete%20Mono.ttf
```
  * Copy the font files to either your system font folder (often /usr/share/fonts/) or user font folder (often ~/.local/share/fonts/ or /usr/local/share/fonts).
```
sudo mv Hack*.ttf /usr/share/fonts/TTF/
```
  * Clear and regenerate your font cache and indexes with the following command:
```
$ fc-cache -f -v
```
  * You can confirm that the fonts are installed with the following command:
```
$ fc-list | rg "Hack"
```

* Open `gnome-tweaks` and set
  * Fonts >> Interface Text: Hack Nerd Font Regular 11
  * Fonts >> Document Text: Hack Nerd Font Regular 11
  * Fonts >> Monospace Text: Hack Nerd Font Mono Regular 10
  * Fonts >> Legacy Window Titles Text: Hack Nerd Font Bold 11

* Pull my dotfiles
```
mkdir ~/Documents/code
cd ~/Documents/code
git clone git@github.com:pierrechevalier83/dotfiles
```

* Configure git
```
> ln -s ~/Documents/code/dotfiles/git/.gitconfig ~/.gitconfig
```
* Configure zsh
```
> ln -s ~/Documents/code/dotfiles/zsh/.zshrc ~/.zshrc
> zsh # configure p10k by following the prompt
```
* Configure neovim
```
> sudo pacman -S nodejs npm # Needed for coc plugin
> mkdir ~/.config/nvim
> curl -fLo ~/.local/share/nvim/site/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
> ln -s ~/Documents/code/dotfiles/neovim/init.vim ~/.config/nvim/init.vim
> nvim
> :PlugInstall
> :UpdateRemotePlugins
> :CocInstal coc-rust-analyzer
```
* Configure alacritty
```
mkdir ~/.config/alacritty
ln -s ~/Documents/code/dotfiles/alacritty/alacritty.yml ~/.config/alacritty/alacritty.yml
```
```
sudo pacman -S brightnessctl
```
* Customize some shortcuts
  * Open gnome-settings
  * Keyboard >> Keyboard Shortcuts >> Customize shortcuts
    * Windows >> Close window: Super+q
	* Navigation
	  * Move to workspace on the left: Ctrl+Alt+Up
	  * Move to workspace on the right: Ctrl+Alt+Down
	  * Move window one workspace to the left: Shift+Ctrl+Alt+Up
	  * Move window one workspace to the right: Shift+Ctrl+Alt+U    * Sound and media (as appropriate based on keyboard)
	  * Volume down
	  * Volume up
	  * Volume mute/unmute
	* Windows
	  * Close window: Super+q
	* Custom Shortcuts
	  * alacritty: Super+Return
	  * brightness down: `brightnessctl -c backlight set 2%-`
	  * brightness up `brightnessctl -c backlight set 2%+`
	  * firefox: Super+f
	  * keyboard brightness down: `brightnessctl -d *::kbd_backlight set 10%-`
	  * keyboard brightness up `brightnessctl -d *::kbd_backlight set 10%+`

* Configure [gdm](https://wiki.archlinux.org/title/GDM)
```
> sudo mkdir /etc/dconf/profile
> sudo nvim /etc/dconf/profile/gdm
```
```
/etc/dconf/profile/gdm
---
user-db:user
system-db:gdm
file-db:/usr/share/gdm/greeter-dconf-defaults
```
```
> sudo mkdir /etc/dconf/db/gdm.d
> sudo nvim /etc/dconf/db/gdm.d/06-tap-to-click
```
```
/etc/dconf/db/gdm.d/06-tap-to-click
---
[org/gnome/desktop/peripherals/touchpad]
tap-to-click=true
```
```
sudo dconf update
```
* In settings, under Users,
  * pick a profile picture for your user in gdm
  * Enable autologin
* Enable [bluetooth support](https://wiki.archlinux.org/title/Bluetooth)
```
sudo pacman -S bluez
sudo systemctl enable bluetooth
sudo systemctl start bluetooth
```

# TODO:
* replace beginning, especially systemd-firstboot with arch_install
* Setup btrfs snapshots management
* Setup printers
* install and configure sway
