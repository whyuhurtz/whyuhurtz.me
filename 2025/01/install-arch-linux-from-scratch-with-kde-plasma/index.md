# Install Arch Linux from Scratch with KDE Plasma


{{< param description >}}

# 1. Pre-installation

## 1.1. Set Console Display Font

```bash
setfont ter-132n
```

## 1.2. Check Firmware Boot Mode (BIOS/UEFI)

```bash
ls /sys/firmware/efi/efivars
```

- If the output shows an error: `no such file or directory`, then the firmware boot mode of your hardware is **BIOS**. Otherwise, you'll have to create **ESP** (EFI System Partition) later.

- To make sure which GRUB Bootloader you should use later, you can use the command below.

```bash
cat /sys/firmware/efi/fw_platform_size
```

- If the output shows **64**, it means that you can use any boot loader that you like. Therefore, if the output is **32**, you've only 2 choices: `grub` or `systemd-grub`.
- For more info, read the documentation page: [https://wiki.archlinux.org/title/Boot_loader](https://wiki.archlinux.org/title/Boot_loader).

## 1.3. Connect to the Internet

- Check internet connection with the `ping` command.

```bash
ip link # to see a list of interface/network devices that are embedded on your device.
ping archlinux.org to check if you are connected to the internet/not.
```

- Use `iwctl` to connect to the internet via WiFi.

```bash
iwctl # enter iwd daemon.
station wlan0 list # to see the list of SSIDs around you.
station wlan0 connect "SSID_NAME" # connect to your SSID.
# Type your password.
quit
```

## 1.4. Partitioning

- Here, I've a virtual disk with a total size of **50GB** and the virtual disk name is `/dev/vda`.
- I'll use the partition schemas.

| File System | Partition | Size  | Mount Point | BTRFS Subvolumes |
| ----------- | --------- | ----- | ----------- | ---------------- |
| FAT32       | /dev/vda1 | 1GiB  | /boot       | -                |
| BTRFS       | /dev/vda2 | 49GiB | /           | @                |

### 1.4.1. Create New Partition

- Check available disk size.

```bash
lsblk

NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0   7:0    0 820.6M  1 loop /run/archiso/airootfs
sr0    11:0    1   1.1G  0 rom  /run/archiso/bootmnt
vda   254:0    0    50G  0 disk
```

- Enter the `fdisk` interactive mode. Then, I'll create 2 different types of partitions, which are `/dev/vda1` for the boot partition and `/dev/vda2` for the root partition.

```bash
# Enter the fdisk interactive mode.
fdisk /dev/vda

# Type `n` to create new partitions.
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)

# Type `p` for select primary partitions.
Select (default p): p

# The default selected primary partition is `1`.
Partition number (1-4, default 1): 1

# Just press `ENTER` for the first sector.
First sector (2048-104857599, default 2048):

# For the last sector, I adjust the partition size for /dev/vda1 to 1 1GiB.
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-104857599, default 104857599): +1G

Created a new partition 1 of type 'Linux' and of size 1 GiB.

# Repeat the step above to create the 2nd partition.
# For /dev/vda2 or root partition, I'll use the rest of virtual disk size, which is 49GiB.
Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2):
First sector (2099200-104857599, default 2099200):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2099200-104857599, default 104857599):

Created a new partition 2 of type 'Linux' and of size 49 GiB.

# Save the current partition layout and exit.
# Just type `w`.
Command (m for help): w

# Press `p` to print the current partition layout.
```

- **Important**, reload the daemon system after partitioning.

```bash
udevadm settle; systemctl daemon-reload
```

### 1.4.2. Format The Partition with a Specific File System

- After 2 partitions have been created, next we need to format the partitions, so they can be used to store data.
- For the `/dev/vda1` or `/boot` partition, I'll format it to the `FAT32` file system.
- Then, for the `/dev/vda2` or `/` partition, I'll format it to `BTRFS`.

```bash
mkfs.fat -F32 /dev/vda1 # For boot partition.
mkfs.btrfs /dev/vda2 # For root partition.
```

- Check if both partitions were formatted successfully.

```bash
lsblk -f
NAME   FSTYPE   FSVER            LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                                     0   100% /run/archiso/airootfs
sr0    iso9660  Joliet Extension ARCH_202501 2025-01-01-08-45-10-00                     0   100% /run/archiso/bootmnt
vda
├─vda1 vfat     FAT32           697A-BD30
└─vda2 btrfs                    0ab075a0-211d-49de-8eab-3881b581430c
```

### 1.4.3. Mount Temporary File System to `/mnt` Directory

- Mount the created partitions to the correct mount points temporarily. In this case, `/dev/vda1` will be mounted to the `/boot` directory, and `/dev/vda2` will be mounted to the `/` directory.
- Before that, we need to create the `/boot` directory first under the `/mnt` directory.

```bash
mkdir -p /mnt/boot
```

- Then, we can mount the partitions to their mount points.

```bash
mount /dev/vda1 /mnt/boot # Mount boot parition to /boot dir.
mount /dev/vda2 /mnt # Mount root partition to / dir (top hierarchy).
```

- Check if the partitions were mounted successfully.

```bash
lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 820.6M  1 loop /run/archiso/airootfs
sr0     11:0    1   1.1G  0 rom  /run/archiso/bootmnt
vda    254:0    0    50G  0 disk
├─vda1 254:1    0     1G  0 part /mnt/boot
└─vda2 254:2    0    49G  0 part /mnt
```

## 1.5. Install Essential Packages in the Chroot Environment

```bash
# 1. Base system, such as kernel, etc.
pacstrap -K /mnt base base-devel linux linux-firmware sudo

# 2. Networking stuffs
pacstrap -K /mnt dhcp dhclient dhcpcd networkmanager iwd wpa_supplicant wireless_tools netctl net-tools

# 3. Hardware connectivity.
pacstrap -K /mnt alsa-utils bluez bluez-utils blueman man man-db dialog ifplugd cups

# 3.1. Pipewire
pacstrap -K /mnt pipewire wireplumber pipewire-audio pipewire-alsa pipewire-pulse

# 4. Graphics driver (open-source).
pacstrap -K /mnt xorg

# 4.1. NVIDIA GPU driver.
pacstrap -K /mnt nvidia nvidia-settings xf86-video-nouveau

# 4.2. Newer AMD GPU driver.
pacstrap -K /mnt xf86-video-amdgpu

# 4.3. Legacy Radeon GPU driver, like HD7xxx & below.
pacstrap -K /mnt xf86-video-ati

# 4.4. Dedicated Intel graphics.
pacstrap -K /mnt xf86-video-intel intel-media-driver libva-intel-driver libva-mesa-driver mesa vulkan-intel

# 5. Additional packages.
vim wget curl git gcc clang g++ gdb make cmake neofetch smartmontools htop openssh ufw screen cockpit
```

## 1.6. Generate `/etc/fstab` File for Persistent Mounting

- Generate the `/etc/fstab` file in the chroot environment.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

## 1.7. Enter The Chroot Environment

```bash
arch-chroot /mnt
```

# 2. Configure System

## 2.1. Configure Time Zone

- Change the **region** and **city** that you live in.

```bash
ln -sf /use/share/zoneinfo/Asia/Jakarta /etc/localtime
timedatectl set-ntp true
hwclock --systohc
```

## 2.2. Localization (System Language)

- Edit `locale.gen` file.

```bash
vim /etc/locale.gen
# Search and uncomment `en_US.UTF-8 UTF-8` section
# :wq for save and exit.
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

- Set locale config file with `en_US.UTF-8`.

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

- Then, generate localization.

```bash
locale-gen
```

## 2.3. Network Configuration

- Set the hostname for Arch Linux.

```bash
echo "archyucry" > /etc/hostname
```

- Edit the `/etc/hosts` file.

```bash
cat << EOF > /etc/hosts
127.0.0.1       localhost
::1             localhost
127.0.1.1       archyucry.localdomain archyucry
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
EOF
```

## 2.4. Create New User

- Create a new user.

```bash
useradd -G wheel,audio,video,input,dhcpcd,bluetooth -m hurtz1nside
passwd --stdin hurtz1nside
# Type the user password
```

- Configure the `/etc/sudoers` file.

```bash
echo "%wheel ALL=(ALL:ALL) ALL" >> /etc/sudoers.d/wheel
```

## 2.5. Create New Boot Loader

- Install boot loader, in this case I'll be using the `grub` boot loader, because it's very common.

```bash
sudo pacman -Sy grub
```

- Install GRUB boot loader to the virtual disk, which is `/dev/vda`. This only works on BIOS legacy, because I'm using a virtual machine here.

```bash
grub-install --target=i386-pc /dev/vda

# For the EFI system partition, I think you should use the command below.
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=arch
# Set the /boot/efi directory to your correct EFI system partitions mount point.
```

- Configure the GRUB boot loader.

```bash
# Enter the grub default config file.
vim /etc/default/grub

# Uncomment the `GRUB_DISABLE_OS_PROBER=false` in the `/etc/default/grub` file, so other bootable partitions will be detected.
GRUB_DISABLE_OS_PROBER=false
```

- Apply the GRUB boot loader configuration.

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## 2.6. Enable Some Services and Reboot

- Enable some important services before reboot.

```bash
systemctl enable dhcpcd.service
systemctl enable NetworkManager.service
systemctl enable bluetooth.service
systemctl enable cups.service
```

- Exit from the chroot environment, umount the `/mnt` directory, and reboot.

```bash
exit
umount /mnt
systemctl reboot
```

# 3. Post-Install

## 3.1. Connect Arch to the Internet

- Get a DHCP (dynamic) IP address to connect to the internet.

```bash
sudo dhcpcd enp1s0

# Check the internet connection with ping.
ping google.com
```

## 3.2. Install KDE Plasma Desktop Environment

- After successfully connecting to the internet, we can now install KDE Plasma.

```bash
# Install some required utilities.
sudo pacman -Sy plasma konsole dolphin ark kwrite kcalc spectacle krunner partitionmanager parted packagekit-qt5

# Install display manager.
sudo pacman -Sy sddm

# Install some GUI apps.
sudo pacman -Sy firefox gedit vlc terminator
```

- Enable the `sddm` display manager service, then reboot.

```bash
sudo systemctl enable sddm.service
reboot
```

## 3.3. Extra: Install Yay (AUR Helper)

- Open Konsole / Terminal, then copy this script.

```bash
cd ~/Downloads/
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
```

# References

- https://wiki.archlinux.org/title/Installation_guide
- https://github.com/XxAcielxX/arch-plasma-install
- https://forums.debian.net/viewtopic.php?t=155410 (Issue copy-paste Virt-Manager)

