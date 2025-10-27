# Install Ubuntu GNOME from Scratch using debootstrap


{{< param description >}}

# Pre-installation

- Prepare the installation medium. I recommend that you download the Ubuntu Desktop ISO version, because it has Firefox installed by default, which you can use for copy and paste or browsing.
- Boot into Ubuntu and open a new Terminal.
- Plug the Ethernet (LAN) cable into your PC/Laptop and get a dynamic IP address (or you can use wireless either).
- After the LAN cable is plugged in, test if it's connected to the internet.

```bash
ping ubuntu.com
```

# Install Required Tools

- Add Ubuntu `universe` APT repository.

```bash
sudo -i # Change to root (super user).
apt-add-repository universe
# Then, press <ENTER> to continue.
```

- Update the repository and install some required tools, such as `vim`, `debootstrap`, etc.

```bash
# Update and install some required tools.
sudo apt update && sudo apt install -y vim \
	debootstrap arch-install-scripts cryptsetup
```

# Console Display

- Set console display to **Terminus** with **16x32** font size.

```bash
dpkg-reconfigure console-setup
```

# System Clocks

- After connecting to the internet, you can update the system clock according to your location.

```bash
# Set time zone according to your location
timedatectl set-timezone Asia/Jakarta
# or manually
sudo ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
```

# Partitioning

> **Note**: Here I've 256GB SSD SATA storage for this demonstration.

- If any partitioning is available before, just remove or clean all the partitions.

```bash
sfdisk --delete /dev/sda
dd if=/dev/zero of=/dev/sda bs=512 count=10000
```

- Disable swap if any swap partition is enabled by default.

```bash
swapoff -a
```

- After the previous partitions are cleaned, the next step is to do partitioning manually from scratch.
- Here are the partition layouts/schemas that I want.

| Partition Name                      | Size                             | Filesystem              | Used for                | BTRFS Subvolumes | Encrypted            |
| ----------------------------------- | -------------------------------- | ----------------------- | ----------------------- | ---------------- | -------------------- |
| /dev/sda1<br>/dev/sda2<br>/dev/sda3 | +1024MiB<br>+100GiB<br>+137.5GiB | FAT32<br>BTRFS<br>BTRFS | /boot<br>/<br>/home<br> | -<br>@<br>@home  | no<br>luks2<br>luks2 |

```bash
# Enter the fdisk interactive mode
fdisk /dev/sda

# Define the GPT partition label
# or DOS if not supported
g # gpt

# Create first partition /dev/sda1 for EFI
n       # create new partition
<ENTER> # default selected partition
<ENTER> # default start sector
+1G     # allocate 1GiB for EFI system partition

# Change the first partition type
t       # change partition type
1       # Select 1 as EFI system partition type

# Create second partition /dev/sda2 for root
n
2       # select id 2 for the second partition
<ENTER> # default start sector
+100G   # allocate 100GiB for root partition
# By default, the partition type is 'Linux filesystem'

# Create third partition /dev/sda3 for home
n
3       # select id 3 for the third partition
<ENTER> # default start sector
<ENTER> # allocate the rest of the available disk size

# Write the current partition layout and quit
w
```

- If you're dual-booting with another operating system like Windows or macOS, it's recommended that you **disable the boot flag** on that particular installation drive.

```bash
# Disable boot flag on Windows
# Do this if you're in dual-boot mode
parted /dev/nvme0n1
print
set 1 boot off
quit
```

- Reload the system daemon to apply partitions.

```bash
udevadm settle; systemctl daemon-reload
```

- Make sure all partitions are configured correctly.

```bash
lsblk
```

# Encrypt Partitions with LUKS2

- Encrypt the root and home data partitions with LUKS.

```bash
cryptsetup --type luks2 luksFormat /dev/sda2 # root
cryptsetup --type luks2 luksFormat /dev/sda3 # home

# Make a memorable passphrase for both data partitions
```

- Decrypt the data partitions to be able to create Btrfs subvolumes and mount the partitions.

```bash
cryptsetup luksOpen /dev/sda2 crypt_system  # root
cryptsetup luksOpen /dev/sda3 crypt_home    # home
```

- Now both data partitions are successfully encrypted with LUKS, which are located at `/dev/mapper/crypt_system` (`/dev/sda2`) and `/dev/mapper/crypt_home` (`/dev/sda3`).

- Next, generate a new `keyfile` to automatically unlock the home partition (`/dev/sda3`).

```bash
# Create a new keyfile for the home data partition
dd if=/dev/urandom of=/root/keyfile bs=1024 count=4

# Make an appropriate permission for that keyfile
chmod 0400 /root/keyfile
```

- Add the generated `keyfile` to the encrypted home partition.

```bash
cryptsetup luksAddKey /dev/sda3 /root/keyfile
```

- Verify the encrypted data partitions with `luksDump` options.

```bash
cryptsetup luksDump /dev/sda2
cryptsetup luksDump /dev/sda3
```

- Configure the `/etc/crypttab` file to define which data partitions are encrypted with LUKS.

```bash
# Define the encrypted root partition
echo "crypt_system UUID="$(cryptsetup luksDump /dev/sda2 | grep UUID | awk '/UUID/ { print $2 }')" none luks" >> /etc/crypttab

# Define the encrypted home partition
echo "crypt_home UUID="$(cryptsetup luksDump /dev/sda3 | grep UUID | awk '/UUID/ { print $2 }')" /root/keyfile luks" >> /etc/crypttab
```

# Formatting Partition

```bash
# Create boot filesystem
mkfs.fat -F32 /dev/sda1

# Create encrypted home and root filesystem
mkfs.btrfs /dev/mapper/crypt_system # /dev/sda2 or root
mkfs.btrfs /dev/mapper/crypt_home   # /dev/sda3 or home
```

# Create BTRFS Subvolumes

```bash
# Create a Btrfs subvolume for encrypted root partitions
mount /dev/mapper/crypt_system /mnt
btrfs subvolume create /mnt/@
umount /mnt

# Create a Btrfs subvolume for the encrypted home partition
mount /dev/mapper/crypt_home /mnt
btrfs subvolume create /mnt/@home
umount /mnt
```

# Mount Boot Partition and BTRFS Subvolumes

```bash
# Mount root BTRFS subvolume to mount point
mount -t btrfs -o noatime,ssd,autodefrag,compress=zstd:1,space_cache=v2,discard=async,subvol=@ /dev/mapper/crypt_system /mnt

# Create directories for home and boot mount points
mkdir -p /mnt/{boot,home}

# Mount home BTRFS subvolume to home mount point
mount -t btrfs -o noatime,ssd,autodefrag,compress=zstd:1,space_cache=v2,discard=async,subvol=@home /dev/mapper/crypt_home /mnt/home

# Mount the boot partition to the boot mount point
mount -o nosuid,nodev,relatime,errors=remount-ro /dev/sda1 /mnt/boot
```

# Prepare Chroot Environment

- Install Ubuntu minimal system with `debootstrap`.

```bash
# Use the fastest mirror while installing the Ubuntu minimal system
debootstrap noble /mnt https://kartolo.sby.datautama.net.id/ubuntu/
```

- Ignore some packages.

```bash
cat <<EOF >> /mnt/etc/apt/preferences.d/ignored-packages
Package: snapd cloud-init landscape-common popularity-contest ubuntu-advantage-tools
Pin: release *
Pin-Priority: -1
EOF
```

- Configure Ubuntu chroot APT sources list.

```bash
vim /mnt/apt/sources.list
# ---- EDIT YOUR APT REPO SOURCE LIST HERE ----
deb https://kartolo.sby.datautama.net.id/ubuntu noble main restricted universe multiverse
deb https://kartolo.sby.datautama.net.id/ubuntu noble-security main restricted universe multiverse
deb https://kartolo.sby.datautama.net.id/ubuntu noble-backports main restricted universe multiverse
deb https://kartolo.sby.datautama.net.id/ubuntu noble-updates main restricted universe multiverse
# ---- EDIT YOUR APT REPO SOURCE LIST HERE ----
```

- Copy the `resolv.conf` file to the chroot environment. It will enable our chroot environment to connect to the internet.

```bash
cp /etc/resolv.conf /mnt/etc/
```

- Copy the `/etc/crypttab` file to the chroot environment.

```bash
cp /etc/crypttab /mnt/etc/
```

- Copy the generated `keyfile` to the chroot environment.

```bash
cp /root/keyfile /mnt/root/
```

# Generate Mounting Table

- Generate the mounting table.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

- Check the `/mnt/etc/fstab` file.

```bash
cat /mnt/etc/fstab
```

# Chroot into Mounted File System

- Chrooting to the mounted file system in the `/mnt` mount point.

```bash
# Change to chroot environment with arch-install-scripts
arch-chroot /mnt

# Or manually with the default chroot command
for d in sys dev proc run tmp; do \
	mount --rbind /$d /mnt/$d && \
	mount --make-rslave /mnt/$d; \
done
chroot /mnt /bin/bash
```

- Create a new file named `/etc/kernel-img.conf` and put the following content. This file is to fix an error **_failed to create symlink to vmlinuz_** when installing Linux kernel image and Linux kernel headers ([_read more_](https://askubuntu.com/questions/1275595/issue-with-installing-the-latest-linux-image-in-lubuntu-20-04)).

```bash
cat <<EOF >> /etc/kernel-img.conf
do_symlinks=no
no_symlinks=yes
EOF
```

- Make sure that in our chroot environment, we can connect to the internet.

```bash
ping ubuntu.com
```

# Install Tools & Software's

- Update Ubuntu APT repository and upgrade installed software.

```bash
apt update && apt upgrade -y
```

- Install kernel image and headers, and some important utilities to make the system run correctly.

```bash
# Install necessary packages
apt install -y linux-{image,headers}-generic cryptsetup \
  linux-firmware btrfs-progs zstd network-manager \
  grub-efi-amd64 efibootmgr
```

> **Note**: Use `grub-pc` if your partition table only supports `msdos` or is still using **BIOS legacy**.

- Install the tools that are needed.

```bash
apt install -y sudo bash bash-completion vim gawk git \
	curl wget man-db parted ntp timeshift fdisk net-tools \
	openssh-client intel-microcode parted dmidecode patch \
	dhcpcd firewalld nftables htop screen neoftech acpid \
	gcc c++ clang gdb make cmake ninja-build xrdp \
	software-properties-common libfuse2t64 libgtk-3-dev \
	pciutils xclip xsel wl-clipboard bat cpu-checker \
	build-essential ca-certificates apt-transport-https
```

- Install audio, power, and multimedia tools.

```bash
apt install -y pipewire pavucontrol cups ffmpeg \
	vlc v4l-utils
```

- Install some fonts that are needed.

```bash
apt install -y fonts-dejavu fonts-ubuntu \
	fonts-jetbrains-mono fonts-firacode
```

- Install some GUI apps that are needed.

```bash
apt install -y evince file-roller gedit cheese
```

- Install Flatpak, plugins for Ubuntu GNOME desktop, and the flathub repo.

```bash
apt install -y flatpak gnome-software-plugin-flatpak && \
	flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

- Install a web browser, I personally choose the Brave browser.

```bash
curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg && \
	curl -fsSLo /etc/apt/sources.list.d/brave-browser-release.sources https://brave-browser-apt-release.s3.brave.com/brave-browser.sources && \
	apt update && apt install -y brave-browser
```

# Configure Time Zone, Locales, and Console Display

- Configure time zone.

```bash
dpkg-reconfigure tzdata
# --- Or manually set time zone with symlink ---
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
```

- Configure locales.

```bash
dpkg-reconfigure locales
# --- Or manually set locales ---
## Edit locale generator file.
vim /etc/locale.gen
## Search and uncomment `en_US.UTF-8 UTF-8` section

## Generate locale
locale-gen

## Create locale config file
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vsconsole.conf
```

- Configure console display to **Terminus** with **16x32** font size.

```bash
dpkg-reconfigure console-setup
# Set console display to Terminus with 16x32
```

# Setting Hostname and Hosts File

- Setting the `/etc/hostname` file.

```bash
echo "ubzz" > /etc/hostname
```

- Setting the `/etc/hosts` file.

```bash
sed -i "2i 127.0.1.1       ubzz" /etc/hosts
```

# Create New User

```bash
useradd -m whyuhurtz -s /bin/bash -G sudo,audio,video,input
passwd whyuhurtz
```

# Network

```bash
# Make sure network-manager utils was installed.
apt install -y network-manager

# Create a file to configure the network via NetworkManager (nmcli)
cat <<EOF > /etc/netplan/network-manager.yaml
network:
  version: 2
  renderer: NetworkManager
EOF
```

# GRUB Boot Loader Config

- Regenerate the `/etc/default/grub` config file.

```bash
# --- BIOS legacy ---
grub-install /dev/sda
# --- UEFI ---
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Ubuntu

update-grub
```

# Reboot into The New System

```bash
exit # from chroot environment
umount /mnt
reboot
```

> **Note**: Make sure all of the configurations in your `/etc/fstab` file are correct before rebooting.

---

# Configure ZRAM Swap

> Note: After successfully getting a new Ubuntu setup, log in with the regular user that we've created before.

- Check internet connection.

```bash
ping ubuntu.com
```

- Install the zram swap management utility.

```bash
# Install zram-tools
sudo apt install -y zram-tools
```

- Configure zram swap.

```bash
# Set zram to use the zstd algorithm compression,
# and use 30% of all free RAM space
cat <<EOF > /etc/default/zramswap
ALGO=zstd
PERCENT=30
EOF
```

- Enable **zramswap** service.

```bash
sudo systemctl enable zram
```

> **Note**: If you're running a different OS on a separate SSD, you need to enable the bootable flag again.

```bash
sudo parted /dev/nvme0n1
set 1 boot on
exit
sudo udevadm settle; sudo systemctl daemon-reload
```

- Reboot the system to apply the zram swap configuration.

```bash
sudo reboot
```

# Install GNOME Desktop Environment

> Note: You've 2 options here, install Ubuntu minimal desktop or with full applications installed. I personally choose the fully installed version.

```bash
# Install full Ubuntu GNOME desktop
sudo apt install -y gdm3 ubuntu-gnome-desktop \
	gnome-tweaks gnome-backgrounds \
	gnome-shell-extension-manager

# Or install a minimal Ubuntu GNOME desktop
sudo apt install -y ubuntu-desktop-minimal
```

# Start Some Services and Reboot

```bash
# Enable some important services
sudo systemctl enable --now bluetooth.service
sudo systemctl enable --now NetworkManager.service
sudo systemctl enable --now cups.service
sudo systemctl enable --now acpid.service
sudo systemctl enable --now nftables.service

# Reboot the system
sudo reboot
```

# References

- https://blog.scheib.me/2023/05/01/debootstrapping-debian.html
- https://www.craftware.info/projects-lists/faster-linux-on-low-memory-using-zram-ubuntu-22-04/

