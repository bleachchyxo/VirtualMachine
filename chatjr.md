# intro

The way i install my OS is by booting my pc from a flash drive with the artix linux (arch based) iso burnt. once on the artix terminal i clone
my git repo and run this script as root

    #!/bin/bash
    set -euo pipefail
    
    # Check if the script is run as root
    if [[ $EUID -ne 0 ]]; then
      echo "Please run as root."
      exit 1
    fi
    
    firmware="BIOS"
    [ -d /sys/firmware/efi ] && firmware="UEFI"
    
    # Function to print colored messages
    message() {
      local color="$1"
      local msg="$2"
      case "$color" in
        green)  echo -e "\033[32m[+]\033[0m $msg" ;;
        yellow) echo -e "\033[33m[+]\033[0m $msg" ;;
        blue)   echo -e "\033[34m[+]\033[0m $msg" ;;
        *)      echo "[+] $msg" ;;
      esac
    }
    
    # Prompt with default value
    default_prompt() {
      local prompt="$1"
      local default="$2"
      read -rp "$prompt [$default]: " input
      echo "${input:-$default}"
    }
    
    # Yes/No confirmation prompt with default answer
    confirmation() {
      local prompt="$1"
      local default="${2:-no}"
      local yn_format
      [[ "${default,,}" =~ ^(yes|y)$ ]] && yn_format="[Y/n]" || yn_format="[y/N]"
      read -rp "$prompt $yn_format: " answer
      answer="${answer:-$default}"
      [[ "${answer,,}" =~ ^(yes|y)$ ]] || { echo "Aborted."; exit 1; }
    }
    
    echo "DarkArtix Installer v0.1"
    echo "Firmware: $firmware"
    
    # Choosing a disk to install
    message blue "Choosing a disk"
    echo "Available disks:"
    mapfile -t available_disks < <(lsblk -dno NAME,SIZE,TYPE | awk '$3=="disk" && $1!~/loop|ram/ {print $1, $2}')
    ((${#available_disks[@]})) || { echo "No disks detected."; exit 1; }
    
    max_name_length=0
    max_size_length=0
    
    for disk_entry in "${available_disks[@]}"; do
      disk_name="${disk_entry%% *}"
      disk_size="${disk_entry#* }"
    
      (( ${#disk_name} > max_name_length )) && max_name_length=${#disk_name}
      (( ${#disk_size} > max_size_length )) && max_size_length=${#disk_size}
    done
    
    for disk_entry in "${available_disks[@]}"; do
      disk_name="${disk_entry%% *}"
      disk_size="${disk_entry#* }"
      disk_path="/dev/$disk_name"
    
      partition_type=$(lsblk -dn -o PTTYPE "$disk_path")
      disk_model=$(fdisk -l "$disk_path" 2>/dev/null | awk -F: '/Disk model/ {gsub(/^ +/,"",$2); print $2}')
    
      printf "  %-${max_name_length}s  %-${max_size_length}s (%s)\n" \
        "$disk_name" "$disk_size" "${disk_model:-$partition_type}"
    done
    
    default_disk="${available_disks[0]%% *}"
    disk_choice=$(default_prompt "Choose a disk to install" "$default_disk")
    disk_path="/dev/$disk_choice"
    
    if [[ -z "${disk_choice:-}" || ! -b "$disk_path" ]]; then
    echo "Invalid or missing disk selection."
    exit 1
    fi
    
    if [[ "$disk_path" =~ (nvme|mmcblk) ]]; then
    part_prefix="p"
    else
    part_prefix=""
    fi
    
    boot_partition="${disk_path}${part_prefix}1"
    root_partition="${disk_path}${part_prefix}2"
    home_partition="${disk_path}${part_prefix}3"
    
    confirmation "This will erase all data on $disk_path. Continue?" "no"
    
    # setting region
    message blue "Setting the region"
    zone_root="/usr/share/zoneinfo"
    
    while true; do
      echo "Available continents:"
      echo "Africa  America  Antarctica  Asia  Atlantic  Australia  Europe  Mexico  Pacific  US"
      region="$(tr '[:upper:]' '[:lower:]' <<< "$(default_prompt "Continent" "America")")"
      region="${region^}"
      [[ -d "$zone_root/$region" || -d "$zone_root/${region^^}" ]] || { echo "Invalid option."; continue; }
      region=$( [[ -d "$zone_root/$region" ]] && echo "$region" || echo "${region^^}" )
    
      region_path="$zone_root/$region"
      timezone="$region"
    
      while true; do
        echo "Available cities in $timezone:"
        ls "$region_path"
        cities=($(ls "$region_path"))
        city=$(default_prompt "City/Timezone" "${cities[RANDOM % ${#cities[@]}]}")
    
        chosen_city=""
        for e in "${cities[@]}"; do [[ "${e,,}" == "${city,,}" ]] && chosen_city="$e" && break; done
        [[ -z "$chosen_city" ]] && { echo "Invalid option."; continue; }
    
        region_path="$region_path/$chosen_city"
        timezone="$timezone/$chosen_city"
        [[ -f "$region_path" ]] && break 2
      done
    done
    
    # Hostname and username
    message blue "Hostname and username"
    hostname=$(default_prompt "Hostname" "artix")
    username=$(default_prompt "Username" "user")
    
    # Setting password for root and user
    message blue "Passwords"
    echo "Set root password:"
    while true; do
      read -s -p "Root password: " rootpass1; echo
      read -s -p "Confirm root password: " rootpass2; echo
      [[ "$rootpass1" == "$rootpass2" && -n "$rootpass1" ]] && break || echo "Passwords do not match or are empty. Try again."
    done
    
    # Prompt for user password
    echo "Set password for user '$username':"
    while true; do
      read -s -p "User password: " userpass1; echo
      read -s -p "Confirm user password: " userpass2; echo
      [[ "$userpass1" == "$userpass2" && -n "$userpass1" ]] && break || echo "Passwords do not match or are empty. Try again."
    done
    
    # formatting disk and creating partitions
    disk_name=$(basename "$disk_path")
    total_gb=$(( $(< /sys/block/$disk_name/size) * $(< /sys/block/$disk_name/queue/hw_sector_size) / 1024 / 1024 / 1024 ))
    
    case $total_gb in
      [0-9]) boot_size=0.5 root_size=4 ;;
      1[0-9]) boot_size=0.5 root_size=6 ;;
      2[0-9]|3[0-9]) boot_size=1 root_size=8 ;;
      [4-9][0-9]) boot_size=1 root_size=20 ;;
      1[0-9][0-9]|*) boot_size=1 root_size=30 ;;
    esac
    
    for partition in $(lsblk -ln -o NAME "$disk_path" | tail -n +2); do
        mount_point=$(lsblk -ln -o MOUNTPOINT "/dev/$partition")
        [ -n "$mount_point" ] && umount "/dev/$partition"
    done
    
    if [[ "$firmware" == "UEFI" ]]; then
    fdisk "$disk_path" <<EOF
    g
    n
    1
    
    +${boot_size}G
    t
    1
    n
    2
    
    +${root_size}G
    n
    3
    
    
    w
    EOF
    else
    fdisk "$disk_path" <<EOF
    o
    n
    p
    1
    
    +${boot_size}G
    n
    p
    2
    
    +${root_size}G
    n
    p
    3
    
    
    w
    EOF
    fi
    
    # Wait for kernel to detect new partitions
    udevadm settle
    sleep 2
    
    for p in "$boot_partition" "$root_partition" "$home_partition"; do
        until [[ -b "$p" ]]; do
            sleep 1
        done
    done
    
    # Create filesystems
    if [[ "$firmware" == "UEFI" ]]; then
        mkfs.fat -F32 "$boot_partition"
    else
        mkfs.ext4 -F "$boot_partition"
    fi
    
    mkfs.ext4 -F "$root_partition"
    mkfs.ext4 -F "$home_partition"
    
    # Mount filesystems
    mount "$root_partition" /mnt
    mkdir -p /mnt/{boot,home}
    mount "$boot_partition" /mnt/boot
    mount "$home_partition" /mnt/home
    
    # Install base system
    base_packages=(
        base
        base-devel
        runit
        elogind-runit
        linux
        linux-firmware
        neovim
        networkmanager
        networkmanager-runit
        grub
    )
    
    [[ "$firmware" == "UEFI" ]] && base_packages+=(efibootmgr)
    
    basestrap /mnt "${base_packages[@]}"
    fstabgen -U /mnt >> /mnt/etc/fstab
    
    # Configure installed system
    artix-chroot /mnt /bin/bash <<EOF
    set -e
    
    ln -sf "/usr/share/zoneinfo/$timezone" /etc/localtime
    hwclock --systohc
    
    sed -i 's/^#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
    locale-gen
    
    echo "LANG=en_US.UTF-8" > /etc/locale.conf
    echo "$hostname" > /etc/hostname
    
    cat >> /etc/hosts <<HOSTS
    127.0.0.1 localhost
    ::1 localhost
    127.0.1.1 $hostname.localdomain $hostname
    HOSTS
    
    useradd -m -G wheel "$username"
    
    sed -i \
    's/^# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' \
    /etc/sudoers
    
    ln -sf /etc/runit/sv/NetworkManager \
    /etc/runit/runsvdir/default/NetworkManager
    
    if [[ "$firmware" == "UEFI" ]]; then
        grub-install \
            --target=x86_64-efi \
            --efi-directory=/boot \
            --bootloader-id=GRUB
    else
        grub-install --target=i386-pc "$disk_path"
    fi
    
    grub-mkconfig -o /boot/grub/grub.cfg
    
    echo "root:$rootpass1" | chpasswd
    echo "$username:$userpass1" | chpasswd
    EOF
    
    unset rootpass1 rootpass2 userpass1 userpass2
    
    echo
    message green "Installation complete. Please reboot and remove the installation media."

once rebooted i login into my user and clone again the repo (the repo from earlier was lost since we were booting on the usb chrooting). so far i have
an OS installed but no UI just terminal. thats why we run a script from the repo as root. this is the script;

    #!/bin/bash
    set -euo pipefail
    
    # Ensure script is run with sudo
    if [[ -z "${SUDO_USER:-}" ]]; then
      echo "This script must be run with sudo."
      exit 1
    fi
    
    # Function to print colored messages
    message() {
      local color="$1"
      local msg="$2"
      case "$color" in
        green)  echo -e "\033[32m[+]\033[0m $msg" ;;
        yellow) echo -e "\033[33m[+]\033[0m $msg" ;;
        blue)   echo -e "\033[34m[+]\033[0m $msg" ;;
        *)      echo "[+] $msg" ;;
      esac
    }
    
    # Determine the real user
    USER_DIR="/home/$SUDO_USER"
    
    # Update system and installing packages
    pacman -Syyu --noconfirm
    pacman -S --noconfirm \
      gcc make git \
      patch curl \
      libx11 libxinerama libxft \
      xorg xorg-xinit \
      ttf-dejavu ttf-font-awesome \
      alsa-utils-runit \
      xcompmgr
    
    # .config 
    mkdir -p "$USER_DIR/.config"  # Using $SUDO_USER to point to correct home directory
    
    # Cloning suckless repos with error handling
    for repository in dwm dmenu st; do
      git -C "$USER_DIR/.config" clone --depth 1 "https://git.suckless.org/$repository" "$USER_DIR/.config/$repository" || { echo "Failed to clone $repository"; exit 1; }
    done
    
    # dmenu setup
    sed -i 's/static int topbar = 1;/static int topbar = 0;/' "$USER_DIR/.config/dmenu/config.def.h"
    make -C "$USER_DIR/.config/dmenu" install || { echo "dmenu compilation failed"; exit 1; }
    
    # dwm setup with patches
    curl -s -o "$USER_DIR/.config/dwm/dwm-fullgaps-20200508-7b77734.diff" https://dwm.suckless.org/patches/fullgaps/dwm-fullgaps-20200508-7b77734.diff
    patch -d "$USER_DIR/.config/dwm" < "$USER_DIR/.config/dwm/dwm-fullgaps-20200508-7b77734.diff"
    sed -i 's/static const unsigned int gappx     = 5;/static const unsigned int gappx     = 15;/' "$USER_DIR/.config/dwm/config.def.h"
    sed -i '8s/topbar\s*=\s*1;/topbar = 0;/g' "$USER_DIR/.config/dwm/config.def.h"
    sed -i 's/#define MODKEY Mod1Mask/#define MODKEY Mod4Mask/' "$USER_DIR/.config/dwm/config.def.h"
    make -C "$USER_DIR/.config/dwm" install || { echo "dwm compilation failed"; exit 1; }
    rm $USER_DIR/.config/dwm/*.diff
    rm $USER_DIR/.config/dwm/*.orig
    
    # st setup with patches
    curl -s -o "$USER_DIR/.config/st/st-alpha-20240814-a0274bc.diff" https://st.suckless.org/patches/alpha/st-alpha-20240814-a0274bc.diff
    patch -d "$USER_DIR/.config/st" < "$USER_DIR/.config/st/st-alpha-20240814-a0274bc.diff"
    curl -s -o "$USER_DIR/.config/st/st-blinking_cursor-20230819-3a6d6d7.diff" https://st.suckless.org/patches/blinking_cursor/st-blinking_cursor-20230819-3a6d6d7.diff
    patch -d "$USER_DIR/.config/st" < "$USER_DIR/.config/st/st-blinking_cursor-20230819-3a6d6d7.diff"
    sed -i 's|Liberation Mono:pixelsize=[0-9]*:antialias=true:autohint=true|Liberation Mono:pixelsize=26:antialias=true:autohint=true|' "$USER_DIR/.config/st/config.def.h"
    sed -i 's/XK_Prior/XK_K/; s/XK_Next/XK_J/' "$USER_DIR/.config/st/config.def.h"
    make -C "$USER_DIR/.config/st" install || { echo "st compilation failed"; exit 1; }
    rm $USER_DIR/.config/st/*.diff
    rm $USER_DIR/.config/st/*.orig
    
    # Setting final details
    ln -s /etc/runit/sv/alsa /etc/runit/runsvdir/default/
    cat > "$USER_DIR/.bash_profile" <<'EOF'
    if [[ -z $DISPLAY ]] && [[ $(tty) = /dev/tty1 ]]; then
        startx
    fi
    EOF
    chown "$SUDO_USER:$SUDO_USER" "$USER_DIR/.bash_profile"
    cp "$(dirname "$0")/Files/xinitrc" "$USER_DIR/.xinitrc"
    chown -R "$SUDO_USER:$SUDO_USER" "$USER_DIR/.config"
    chown -R "$SUDO_USER:$SUDO_USER" "$USER_DIR/.xinitrc"
    
    message green "Enviroment succesfully installed. Reboot or type startx."

after type `startx` the enviroment appears so i run the following commands

    sudo pacman -S nvidia-open-dkms linux-headers dkms base-devel
    qemu libvirt libvirt-runit dnsmasq virt-manager
    sudo usermod -aG libvirt,kvm $USER
    sudo ln -s /etc/runit/sv/libvirtd /etc/runit/runsvdir/default
    sudo ln -s /etc/runit/sv/virtlockd /etc/runit/runsvdir/default
    sudo ln -s /etc/runit/sv/virtlogd /etc/runit/runsvdir/default

then reboot again. once again on the system i download the windows 10 iso. i wanna create a virtual machine on kvm hosting a windows 10 system so i run

    sudo virsh net-start default
    
and assing `/dev/sda` for the vm;

    lsblk
    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    sda           8:0    0 465.8G  0 disk
    ├─sda1        8:1    0    50M  0 part
    ├─sda2        8:2    0 465.2G  0 part
    └─sda3        8:3    0   522M  0 part
    nvme0n1     259:0    0 238.5G  0 disk
    ├─nvme0n1p1 259:1    0     1G  0 part /boot
    ├─nvme0n1p2 259:2    0    30G  0 part /
    └─nvme0n1p3 259:3    0 207.5G  0 part /home

once we created the vm i check for updates and make it up to date. but now this is where im stuck. on device manager the windows 10 shows "micorosft basic display adaptor" on `display adapters` and i want to take my 3080 nvidia gpu while the host (artix) runs using IGPU which i enabled it previiously on the BIOS.
be careful last time i had to reinstall my whole OS since i got with no display/blackscreen. also i have a pair of physical speakers idk if i need to do the passthrough also so i listen the vm and then switch them back to the host after finishing using the vm. or is there a way to make the vm produce sounds and hear them through the host as a video frokm yotubue idk. idk how to do idk what to do and idk whats the best efficient and clean weay to do it

     lspci
    00:00.0 Host bridge: Intel Corporation Comet Lake-S 6c Host Bridge/DRAM Controller (rev 03)
    00:01.0 PCI bridge: Intel Corporation 6th-10th Gen Core Processor PCIe Controller (x16) (rev 03)
    00:02.0 VGA compatible controller: Intel Corporation CometLake-S GT2 [UHD Graphics 630] (rev 03)
    00:14.0 USB controller: Intel Corporation 400 Series Chipset Family USB 3.2 Gen 2x1 (10 Gbs) xHCI Host Controller
    00:14.2 RAM memory: Intel Corporation 400 Series Chipset Family Shared SRAM
    00:15.0 Serial bus controller: Intel Corporation 400 Series Chipset Family I2C #0
    00:15.1 Serial bus controller: Intel Corporation 400 Series Chipset Family I2C #1
    00:16.0 Communication controller: Intel Corporation 400 Series Chipset Family HECI #1
    00:17.0 SATA controller: Intel Corporation 400 Series Chipset Family SATA Controller (AHCI) (Desktop)
    00:1b.0 PCI bridge: Intel Corporation 400 Series Chipset Family PCIe Root Port #17 (rev f0)
    00:1c.0 PCI bridge: Intel Corporation 400 Series Chipset Family PCIe Root Port #1 (rev f0)
    00:1c.4 PCI bridge: Intel Corporation 400 Series Chipset Family PCIe Root Port #5 (rev f0)
    00:1d.0 PCI bridge: Intel Corporation 400 Series Chipset Family PCIe Root Port #9 (rev f0)
    00:1f.0 ISA bridge: Intel Corporation Z490 Chipset LPC/eSPI Controller
    00:1f.3 Audio device: Intel Corporation 400 Series Chipset Family HD Audio
    00:1f.4 SMBus: Intel Corporation 400 Series Chipset Platform SMBus
    00:1f.5 Serial bus controller: Intel Corporation 400 Series Chipset Family SPI (flash) Controller
    01:00.0 VGA compatible controller: NVIDIA Corporation GA102 [GeForce RTX 3080] (rev a1)
    01:00.1 Audio device: NVIDIA Corporation GA102 High Definition Audio Controller (rev a1)
    02:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981/PM983
    04:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8111/8168/8211/8411 PCI Express Gigabit Ethernet Controller (rev 15)

also made sure these remain enabled:

    -Intel VT-x
    -Intel VT-d
    -Above 4G Decoding (if available)
    -Integrated Graphics = Enabled
    -Primary Display = IGPU

The monitor  plugged into the motherboard HDMI not the RTX 3080.

edited `/etc/default/grub` and changed;

    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"

to 

    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet intel_iommu=on"

updated the grub

    sudo grub-mkconfig -o /boot/grub/grub.cfg

and rebooted
