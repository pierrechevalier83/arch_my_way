# How to install Arch Linux, my way, but faster

Now to streamline the installation, the Arch Linux Installer Script, a.k.a [alis](https://github.com/picodotdev/alis) is a fantastic tool.

I have encoded many of my preferences for both laptops I am currently using in my [own configuration for alis](https://github.com/pierrechevalier83/alis-conf)

# Download the iso and create a bootable medium:

* [Download](https://archlinux.org/download/) the latest iso.

* With the USB drive inserted, but not mounted, and its name (say `/dev/sdx`) known thanks to `lsblk`.
```
$ dd bs=4M if=path/to/archlinux-version-x86_64.iso of=/dev/sdx conv=fsync oflag=direct status=progress
```

# Connect to wifi (if on bare metal)

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

# Download my alis configuration

* Download alis with my custom configuration and generate a configuration for this specific machine
```
curl -sL https://raw.githubusercontent/pierrechevalier83/alis-conf/main/download.sh | bash
./configure.sh
```
* If specific needs, edit `alis.conf` and `alis-packages.conf` accordingly
* Run the script:
```
./alis.sh
```
* Reboot
```
./alis-reboot.sh
```

# Install and configure the system

Edit /etc/hosts:

```
127.0.0.1   localhost
::1     localhost
127.0.1.1   some_hostname.localdomain   some_hostname
```
# First boot

// TODO: check this is done on next install with updated config

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

// TODO: check gnome-extra (added)

* Setup ssh keys
```
ssh-keygen
```
* Log-in to the github web interface and add the public key (from `~/.ssh/id_rsa.pub`

* Setup rust environment with rustup
```
rustup install nightly stable
rustup default stable
source ~/.cargo/env
```

* Pimp it!
// TODO: check these AUR packages got installed
```
sudo pacman -S arc-gtk-theme arc-icon-theme
```
* Open `gnome-tweaks` and set
  * Appearance >> Applications: Arc Dark
  * Appearance >> Icons: Arc
  * Top Bar >> Battery percentage: true
  * Top Bar >> Weekday: true
  * Windows >> Window Focus: Focus on hover
  * Workspace >> Display handling: Workspaces span displays

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
> sudo ln -s ~/Documents/code/dotfiles/zsh/.zshrc /root/.zshrc
> zsh # configure p10k by following the prompt
> sudo zsh # configure p10k for root by following the prompt (makes sense to style it differently to easily distinguish them)
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

* Enable btrfs snapshots with [snapper](https://wiki.archlinux.org/title/Snapper) and snap-pac
```
sudo pacman -S snapper snap-pac
sudo snapper -c root create-config /
yay -S snapper-gui-git
```
* Backup the boot partition on pacman transactions
  Create this file:
```
/etc/pacman.d/hooks/50-bootbackup.hook
---
[Trigger]
Operation = Upgrade
Operation = Install
Operation = Remove
Type = Path
Target = usr/lib/modules/*/vmlinuz

[Action]
Depends = rsync
Description = Backing up /boot...
When = PostTransaction
Exec = /usr/bin/rsync -a --delete /boot /.bootbackup
```
* Edit the snapper config and keep as many snapshots as desired
```
sudo nvim /etc/snapper/configs/root
```
* Allow easy restores the "proper" way
```
sudo nvim /etc/snapper-rollback.conf
# Change the subvol_snapshots line to
/snapshots
# Change last line to
dev = /dev/mapper/luks
```
* Test the rollback setup at least once end-to-end to avoid bad surprises
```
sudo pacman -S awesome # and keep track of root before that command (say 7)
which awesome # /usr/bin/awesome
sudo snapper-rollback 7
reboot
which awesome # awesome not found
```

# Setup printers

* Setup [printers](https://wiki.archlinux.org/title/CUPS):
  * Setup [avahi](https://wiki.archlinux.org/title/Avahi)
    Edit the file `/etc/nsswitch.conf` and change the hosts line to include `mdns_minimal [NOTFOUND=return]` before resolve and dns:
```
hosts: ... mdns_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] dns ...
  * Open the cups management center (http://localhost:631), login as root
  * Administration >> Add Printer
  * The printer should be present, and its correct driver should be available in the drop-down (EPSON ET-2710 Series, ...)

# TODO:
* install and configure sway
