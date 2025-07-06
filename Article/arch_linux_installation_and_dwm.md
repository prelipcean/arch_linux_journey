# A Step-by-Step Guide to Arch Linux Installation and DWM

As a Linux developer and Arch Linux power user, the **Keep It Simple, Stupid (KISS)** philosophy is central to my work, especially in embedded projects. Arch Linux perfectly embodies this principle, offering unparalleled control, speed, and customization. It's more than just an OS; it's a deep dive into understanding the core of Linux.

This article details a complete Arch Linux installation on a T440 laptop and walks you through the entire process, from the initial bootable USB to a fully customized, minimalist desktop environment using the Dynamic Window Manager (DWM) and other Suckless tools. This isn't just an installation guide; it's a blueprint for creating a system that is truly yours: lean, powerful, and tailored to your exact workflow. Let's dive in.

-----

## Part 1: The Foundation - Installing Arch Linux

### 1. Prepare the Installation Environment

First, we lay the groundwork. This involves verifying your boot mode (UEFI is preferred for modern systems), setting your keyboard layout, and establishing network connectivity. Download the official image from the [Arch Linux Downloads page](https://archlinux.org/download/) and create a bootable device.

**Verify Boot Mode:**

```bash
ls /sys/firmware/efi/efivars
# If variables list, you're in UEFI. An error means BIOS.
```

**Set Keyboard Layout (US Example):**

```bash
loadkeys us
# For others: localectl list-keymaps then loadkeys <layout>
```

**Network Configuration (Wi-Fi Example):**

```bash
iwctl device list                      # List Wi-Fi devices (e.g., wlan0)
iwctl station <device> scan            # Scan for networks
iwctl station <device> get-networks    # List available SSIDs
iwctl station <device> connect "Your_SSID" --passphrase "Your_Wi-Fi_Password"
ping -c 3 archlinux.org                # Verify connectivity
```
> **Tip:** If you encounter issues, check `rfkill list` to ensure your Wi-Fi is not blocked.

**Update System Clock:**

```bash
timedatectl set-ntp true
```

### 2\. Disk Partitioning with LVM (UEFI)

This is where we craft the disk layout. Using LVM (Logical Volume Management) provides flexibility for resizing and managing logical volumes later. We'll set up an EFI System Partition (ESP), a `/boot` partition, and an LVM Physical Volume (PV).

**List Disks:**

```bash
$ lsblk
$ fdisk -l
```

**Create GPT Partition Table:**

```bash
$ gdisk /dev/sda  # Replace sda with your target disk 
o                 # Create new empty GPT partition table 
w; Y              # Write changes and exit
```

**Create Partitions in gdisk:**

```bash
$ gdisk /dev/sda 
n; <CR>; <CR>; +512M; ef00  # EFI System Partition (ESP) - /dev/sda1 
n; <CR>; <CR>; +1G; 8300   # /boot Partition - /dev/sda2 
n; <CR>; <CR>; <CR>; 8e00 # LVM Physical Volume - /dev/sda3 (uses rest of disk) 
p                         # Print partitions to verify
w; Y                      # Write changes and exit
```

**Setup LVM:**

```bash
$pvcreate /dev/sda3                      # Create Physical Volume$ vgcreate vgarch /dev/sda3               # Create Volume Group (vgarch)
$lvcreate -L 8G vgarch -n lvswap         # Create Swap Logical Volume$ lvcreate -L 80G vgarch -n lvroot        # Create Root Logical Volume
$lvcreate -l +100%FREE vgarch -n lvhome  # Create Home Logical Volume$ vgdisplay vgarch
$ lvdisplay vgarch                        # Verify LVM setup
```

**Format Partitions and LVs:**

```bash
$mkfs.fat -F 32 /dev/sda1               # Format ESP$ mkfs.ext4 /dev/sda2                    # Format /boot
$mkswap /dev/mapper/vgarch-lvswap       # Format & create swap$ swapon /dev/mapper/vgarch-lvswap       # Enable swap
$mkfs.ext4 /dev/mapper/vgarch-lvroot    # Format root LV$ mkfs.ext4 /dev/mapper/vgarch-lvhome    # Format home LV
```

**Format Other Disks (Optional):**

```bash
# Example for /dev/sdb as /data
$gdisk /dev/sdb; o; Y; w; Y                  # New GPT table$ gdisk /dev/sdb; n;<CR>;<CR>;<CR>;<CR>; p; w; Y # Single partition
$ mkfs.ext4 -L DATA /dev/sdb1                # Format with label
```

**Mount Partitions:**

```bash
$mount /dev/mapper/vgarch-lvroot /mnt$ mkdir -p /mnt/boot
$mount /dev/sda2 /mnt/boot$ mkdir -p /mnt/boot/efi
$mount /dev/sda1 /mnt/boot/efi$ mkdir -p /mnt/home
$mount /dev/mapper/vgarch-lvhome /mnt/home$ mkdir -p /mnt/data
$mount /dev/sdb1 /mnt/data # Mount your other disk$ lsblk
$ df -h # Verify all mounts
```

### 3\. Base System Installation & Chroot

Install the core Arch Linux packages and `chroot` into the new system to begin configuration.

**Optimize Mirrors:**

```bash
$cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak # Backup$ reflector --country "Romania,Germany,France" --protocol https --sort rate --latest 20 --save /etc/pacman.d/mirrorlist
```

**Install Base System:**

```bash
$ pacstrap /mnt base linux linux-firmware lvm2 base-devel networkmanager
```

**Generate fstab:**

```bash
$genfstab -U /mnt >> /mnt/etc/fstab$ cat /mnt/etc/fstab # Review it!
```

**Chroot into the New System:**

```bash
$ arch-chroot /mnt
```

-----

## Part 2: System Configuration - Bringing Arch to Life

Inside the chroot, we'll configure the essential system settings.

### 1\. Time & Locale

```bash
$ln -sf /usr/share/zoneinfo/Europe/Bucharest /etc/localtime # Set timezone$ hwclock --systohc # Sync hardware clock
$pacman -S vim # Install text editor$ vim /etc/locale.gen
```

**Uncomment in `/etc/locale.gen`:**

```
en_US.UTF-8 UTF-8
```

```bash
$locale-gen # Generate locales$ echo "LANG=en_US.UTF-8" > /etc/locale.conf # Set default locale
```

### 2\. Networking & Hostname

```bash
$ systemctl enable NetworkManager # Enable NetworkManager
$echo "archlaptop" > /etc/hostname # Set hostname$ vim /etc/hosts 
```

**Add in `/etc/hosts`:**

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   archlaptop.localdomain archlaptop
```

### 3\. User Accounts & Sudo

```bash
$ passwd # Set root password
$useradd -m -g users -G wheel,storage,video,audio,network,power -s /bin/bash youruser$ passwd youruser # Set user password
$pacman -S sudo # Install sudo$ EDITOR=vim visudo
```

**Uncomment line in `visudo`:**

```
%wheel ALL=(ALL:ALL) ALL
```

### 4\. Bootloader & Initramfs

These are critical for system boot.

**Install Microcode (Intel Example):**

```bash
$ pacman -S intel-ucode
```

**Configure mkinitcpio (Initramfs):**

```bash
$ vim /etc/mkinitcpio.conf
```

**Add to `HOOKS` line in `/etc/mkinitcpio.conf`:**

```
HOOKS=(... block lvm2 filesystems ...) # lvm2 MUST be before filesystems
```

```bash
$ mkinitcpio -P # Regenerate initramfs
```

**Install & Configure GRUB (UEFI):**

```bash
$pacman -S grub efibootmgr$ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --removable
$ grub-mkconfig -o /boot/grub/grub.cfg
```

> **Note**: If system is BIOS or CPU architecture different than x86\_64 follow the Arch Wiki to install GRUB for BIOS.

### 5\. Finalize & Reboot

```bash
$ exit             # Exit chroot
$umount -R /mnt   # Unmount all partitions$ swapoff -a       # Disable swap
$ reboot           # Remove USB and reboot into your new Arch system!
```

-----

## Part 3: The Minimalist Desktop - DWM & Suckless Tools

Now we build our highly customizable graphical environment.

### 1\. Post-Reboot Verification

Log in as your new user.

**Verify network connectivity:**

```bash
$ping -c 3 8.8.8.8$ sudo systemctl status NetworkManager
```

If NOT active (running) then:

```bash
$ sudo systemctl start NetworkManager
$sudo journalctl -u NetworkManager --since "5 minutes ago" --no-pager$ nmcli radio wifi
```

If disabled then:

```bash
$ sudo nmcli radio wifi on
$ nmcli dev wifi list
$sudo nmcli device wifi connect "Your_SSID_Name" password "Your_Wi-Fi_Password"$ nmcli connection show --active
$ nmcli device status
```

**Check that swap is active:**

```bash
$ swapon --show
```

**Perform a full system update:**

```bash
sudo pacman -Syyu
```

### 2\. Install Core Graphics & Drivers

```bash
$ sudo pacman -S xorg-server                 # Xorg display server 
$ sudo pacman -S mesa libva-mesa-driver      # 3D support and video acceleration for Intel 
$ sudo pacman -S xf86-video-intel            # Intel Xorg driver
```

### 3\. Essential Desktop Utilities & Drivers

```bash
$sudo pacman -S linux-headers                 # Essential for kernel module compilation$ sudo pacman -S git base-devel dialog wget curl htop tree neofetch # Useful CLI tools
$sudo pacman -S unzip p7zip unrar             # Archive tools$ sudo pacman -S ntfs-3g dosfstools mtools     # NTFS and FAT filesystem support
$sudo pacman -S bash-completion               # Enable tab completion$ sudo pacman -S bc                            # Basic calculator
$ sudo pacman -S brightnessctl                 # Brightness control
```

### 4\. Audio & Bluetooth

```bash
$sudo pacman -S pipewire pipewire-alsa pipewire-jack pipewire-pulse alsa-utils wireplumber$ systemctl --user enable pipewire pipewire-pulse wireplumber # Enable user services
$sudo pacman -S bluez bluez-utils pipewire-bluetooth # Bluetooth core and audio support$ sudo systemctl enable --now bluetooth.service
```

### 5\. Fonts for the Perfect Aesthetic

```bash
$sudo pacman -S fontconfig # Core font library$ sudo pacman -S noto-fonts noto-fonts-emoji ttf-dejavu ttf-liberation ttf-ubuntu-font-family 
$ fc-cache -fv
```

For Nerd Fonts, an AUR helper like `yay` is recommended:

```bash
$mkdir ~/builds$ cd ~/builds
$git clone https://aur.archlinux.org/yay.git$ cd yay && makepkg -si
$ yay -S ttf-jetbrains-mono-nerd
```

### 6\. Suckless Toolchain (DWM, ST, Dmenu, Slstatus)

This is where the true customization begins by building from source.

**Clone Repositories & Checkout Tags:**

```bash
$mkdir -p ~/.local/suckless$ cd ~/.local/suckless
$git clone https://git.suckless.org/dwm$ git clone https://git.suckless.org/st
$git clone https://git.suckless.org/dmenu$ git clone https://git.suckless.org/slstatus
```

**For each repo, checkout a stable tag:**

  - e.g., `cd dwm && git checkout 6.5`
  - e.g., `cd st && git checkout 0.9.2`
  - e.g., `cd dmenu && git checkout 5.3`
  - e.g., `cd slstatus && git checkout 1.1`

**Install build dependencies:**

```bash
$ sudo pacman -S libxft libxinerama
```

**Build dwm:**

```bash
$cd ~/.local/suckless/dwm$ cp config.def.h config.h # Always edit config.h, not the default
$ sudo make clean install
```

> Repeat the `cp config.def.h config.h` and `sudo make clean install` process for `st`, `dmenu`, and `slstatus`, customizing their `config.h` files for fonts, colors, and behavior.

### 7\. Customize .xinitrc & Autostart DWM

Your `~/.xinitrc` file launches the graphical session. We will configure it to autostart on TTY1 for a seamless, display-manager-free experience.

**Install Dependencies:**

```bash
$ sudo pacman -S xorg-xrandr picom nitrogen xorg-setxkbmap dunst
```

**Modify `~/.xinitrc`:**

> Remove "\# start some nice programs" from template of xinitrc as we will start our programs. Add at the end of the file:

```bash
# Keyboard layout: setxkbmap us for US layout
setxkbmap us &

# Display resolution: IMPORTANT - "eDP-1" and "1366x768" are examples.
# You MUST verify your actual output name and native resolution with `xrandr` first.
xrandr --output eDP-1 --mode 1366x768 &

# Start slstatus in the background and update dwm's status bar
# The 'while true' loop approach is common for suckless status bars.
while true; do
    xsetroot -name "$(slstatus)"
    sleep 1 # Update every second
done &

# Compositor: picom. The `-b` flag runs it in the background.
picom -b &

# Wallpaper: nitrogen --restore
nitrogen --restore &

# Start a notification daemon (e.g., 'dunst')
# dunst &

# Start a terminal automatically (optional, but convenient for first boot)
st &

# Execute dwm - This MUST be the LAST command in .xinitrc
# The `while true; do dwm > /dev/null 2>&1; done` loop is a robust way to
# automatically restart DWM if it crashes or you exit it with a keybinding.
while true; do
    dwm > /dev/null 2>&1
done
```

**Systemd Autologin for DWM:**

```bash
$ sudo systemctl edit getty@tty1.service
```

Add the following lines in the dedicated section of `tty1.service`:

```ini
[Service]
ExecStart=
ExecStart=-/usr/bin/agetty --autologin yourusername --noclear %I $TERM
```

Add to your `~/.bash_profile` (`$ vim ~/.bash_profile`):

```bash
if [ -z "$DISPLAY" ] && [ "$(tty)" = "/dev/tty1" ]; then
    exec startx
fi
```

-----

## Part 4: Advanced Tools & Management (In Work)

Finally, let's round out the system with essential tools for a modern development workflow.

### 1\. Docker

```bash
$sudo pacman -S docker docker-compose$ sudo systemctl enable --now docker.service
$sudo usermod -aG docker yourusername # Re-login for this to take effect$ docker run hello-world # Test
```

### 2\. Git Configuration

```bash
$git config --global user.name "Your Name"$ git config --global user.email "your.email@example.com"
$git config --global init.defaultBranch main$ git config --global pull.rebase true
$git config --global push.default current$ git config --global core.editor "vim"
$git config --global core.autocrlf false$ git config --global core.longpaths true
```

**Git LFS on Arch Linux:**

```bash
$sudo pacman -S git-lfs$ git lfs install --system # or $ git lfs install --global
```

### 3\. PKM

```bash
# Example for Obsidian (using yay)
$yay -Ss obsidian$ yay -S obsidian

# Example for Logseq (using paru)
$paru -Ss logseq$ paru -S logseq
```

### 4\. Fastfetch (from source)

A sleek and fast system information tool.

```bash
$sudo pacman -S --needed cmake ninja # Build dependencies$ mkdir -p ~/.local/fastfetch-build
$cd ~/.local/fastfetch-build$ git clone https://github.com/fastfetch-cli/fastfetch.git
$ cd fastfetch
$ mkdir build && cd build
$cmake -DCMAKE_INSTALL_PREFIX=/usr -G Ninja ..$ ninja # Compile
$ sudo ninja install # Install
```

### 5\. Neovim (from source)

```bash
$sudo pacman -S base-devel cmake unzip ninja curl git libtool autoconf automake pkgconf gettext$ sudo pacman -S lua npm ripgrep
$git clone https://github.com/neovim/neovim.git$ cd neovim
$ git checkout stable
$make CMAKE_BUILD_TYPE=Release$ sudo make install
$ nvim --version
```

**Updating Neovim later:**

```bash
$ cd neovim
$ git pull
$ make clean
$make CMAKE_BUILD_TYPE=Release$ sudo make install
```

### 6\. Core Desktop Applications

```bash
$sudo pacman -S firefox$ sudo pacman -S thunar
$ sudo pacman -S maim xclip
```

Take a full screenshot with:

```bash
$ maim ~/screenshots/$(date +%F-%H-%M-%S).png
```

-----

## Part 5: Hardening System Security

A default installation is a starting point. Taking proactive steps to secure your system is a critical responsibility for any user.

### 1\. Configure a Firewall

`ufw` (Uncomplicated Firewall) is a user-friendly frontend for managing firewall rules.

**Install UFW:**

```bash
$ sudo pacman -S ufw
```

**Set Defaults & Rules:**

```bash
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow ssh
$ sudo ufw allow http
$ sudo ufw allow https
$ sudo ufw allow from 192.168.1.100 to any port 22
```

**Enable Logging & Firewall:**

```bash
$ sudo ufw logging on
$ sudo ufw enable
$ sudo systemctl enable ufw.service
```

**Check Status:**

```bash
$ sudo ufw status verbose
```

### 2\. Secure SSH Daemon

If you use SSH, it's vital to harden its configuration.

**Disable Root Login & Use Key-Based Authentication:**
Edit `/etc/ssh/sshd_config`:

```ini
PermitRootLogin no
PasswordAuthentication no
```

**Other Secure Configurations:**
Edit `/etc/ssh/sshd_config`:

```ini
AllowUsers yourusername anotheruser
X11Forwarding no
AllowAgentForwarding no
```

After making changes, restart the SSH service:

```bash
$ sudo systemctl restart sshd.service
```

### 3\. Keep Your System Updated

Arch Linux is a rolling release. Regular updates are essential.

```bash
$ sudo pacman -Syu
```

### 4\. Security Auditing

**Lynis:**

```bash
$sudo pacman -S lynis$ sudo lynis audit system
```

> **AppArmor/SELinux:** For environments requiring maximum security, explore Mandatory Access Control (MAC) systems like AppArmor or SELinux.

-----

## Part 6. Miscellaneous

### 1\. Check battery

```bash
$ls /sys/class/power_supply/$ cat /sys/class/power_supply/BAT[NUMBER]/capacity
```

### 2\. Arch User Repository (AUR)

```bash
# Install yay
$sudo pacman -S --needed base-devel git$ git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si && cd .. && rm -rf yay

# Or install paru
$sudo pacman -S --needed base-devel git$ git clone https://aur.archlinux.org/paru.git && cd paru && makepkg -si && cd .. && rm -rf paru
```

### 3\. Set up encryption

```bash
$cryptsetup luksFormat /dev/<partition_number>$ cryptsetup open --type luks /dev/<partition_number> lvm
```

Add `encrypt` to `HOOKS` in `/etc/mkinitcpio.conf` before `lvm2`:

```
HOOKS=(... block encrypt lvm2 filesystems ...)
```

### 4\. Format with BTRFS

BTRFS offers advanced features like snapshots and transparent compression. Here’s a basic setup:

```bash
mkfs.btrfs -f /dev/sdXn
mount /dev/sdXn /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt
mount -o compress=zstd,subvol=@ /dev/sdXn /mnt
mkdir /mnt/home
mount -o compress=zstd,subvol=@home /dev/sdXn /mnt/home
```
Update `/etc/fstab` accordingly.
> **Tip:** See [Arch Wiki: Btrfs](https://wiki.archlinux.org/title/Btrfs) for advanced layouts and snapshot tools.

### 5. Share system clipboard with VIM or NEOVIM

To enable clipboard support, install `xclip` or `xsel` and ensure your Vim/Neovim build includes `+clipboard`:

```bash
$ sudo pacman -s xclip
```
> **Note:** For Neovim, use `"+y` and `"+p` for system clipboard.
> Check clipboard support with `vim --version | grep clipboard`.

### 6. Linux Rice

"Ricing" refers to customizing your Linux desktop’s appearance. For example you can explore themes, icon packs, and dotfiles from [r/unixporn](https://www.reddit.com/r/unixporn/)
> **Tip:** Backup your config files and document your changes.

### 7. Language-Specific Toolchains

Install toolchains and environments for your development needs:

- **Python:** `sudo pacman -S python python-pip`
- **Rust:** `sudo pacman -S rust`
- **Go:** `sudo pacman -S go`
- **Node.js:** `sudo pacman -S nodejs npm`
- **Java:** `sudo pacman -S jdk-openjdk`

> **Tip:** Use virtual environments (e.g., `python -m venv venv`) for Python projects.

-----

Embarking on an Arch Linux journey is a commitment, but the reward is a system that is truly yours, configured to your exact specifications. By following these steps, you've not only installed a powerful operating system but also built a deep understanding of its inner workings – a skillset invaluable for any Linux professional.

**Further Reading:**
- [Arch Wiki - Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [DWM - Suckless](https://dwm.suckless.org/)
- [Arch Wiki - Security](https://wiki.archlinux.org/title/Security)
- [Arch Wiki - General Recommendations](https://wiki.archlinux.org/title/General_recommendations)

\#ArchLinux \#Linux \#DWM \#Suckless \#SystemHardening \#TechTutorial \#OpenSource \#PowerUser \#DIYTech \#Minimalism \#Developer \#Cybersecurity \#SysAdmin
