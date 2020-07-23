# Pre-install 
```sh
timedatectl set-ntp true

# Export values
export USER=waff
export HOSTNAME=nyarch
```

# Partitioning


After checking the devices (i. e. with `lsblk`) values to the following variables
```sh
export DEVNAME_HDD=sda
export DEVNAME_SSD=nvme0n1
export DEVNAME_USB=sdc

export DEV_HDD=/dev/$DEVNAME_HDD
export DEV_SSD=/dev/$DEVNAME_SSD
export DEV_USB=/dev/$DEVNAME_USB
```


### Random data
```sh
dd if=/dev/urandom of=$DEV_HDD bs=4096 status=progress
dd if=/dev/urandom of=$DEV_SSD bs=4096 status=progress
dd if=/dev/urandom of=$DEV_USB bs=4096 status=progress
```

### Partition schemes
device | partition | mount point | type | filesystem | size
------ | --------- | ----------- | ---- | ---------- | ----
usb | 1 | /boot/efi | EFI System | vfat | 512M
usb | 2 | /boot | Linux filesystem | ext4 | 200M

device | partition | mount point | type | filesystem | size
------ | --------- | ----------- | ---- | ---------- | ----
ssd | - | | Free space | - | 1000G

device | partition | mount point | type | filesystem | size
------ | --------- | ----------- | ---- | ---------- | ----
ssd | - | | Free space | - | 1000G

### Partition disks
```sh
# Create partitions
cfdisk $DEV_HDD
cfdisk $DEV_SSD
cfdisk $DEV_USB
```

### Export variables for later use
```sh
export ID_HDD=$(ls -lt /dev/disk/by-id | grep -m 1 $DEVNAME_HDD | awk '{print $9}')
export ID_SSD=$(ls -lt /dev/disk/by-id | grep -m 1 $DEVNAME_SSD | awk '{print $9}')
export ID_USB=$(ls -lt /dev/disk/by-id | grep -m 1 ${DEVNAME_USB}2 | awk '{print $9}')
```


# Disk encryption
### Encrypted boot
```sh
# Encrypt boot
cryptsetup --cipher=twofish-xts-plain64 --hash=sha512 --key-size=512 -i 30000 luksFormat ${DEV_USB}2

# Mount and format boot
cryptsetup open ${DEV_USB}2 cryptboot
mkfs.ext2 /dev/mapper/cryptboot -L $HOSTNAME-boot # ext2 does not have journaling
mkfs.fat -F32 ${DEV_USB}1
mount /dev/mapper/cryptboot /mnt
cd /mnt

# Create random key
dd if=/dev/urandom of=key.img bs=20M count=1

# Encrypt key image
cryptsetup --align-payload=1 --cipher=serpent-xts-plain64  --hash=sha512 --key-size=512 -i 30000 luksFormat key.img
cryptsetup open key.img lukskey
```

### Encrypt drives
```sh
# Create header file
truncate -s 16M header.img

# Set some options
export KEY_SIZE=8192 # max 8192
export OFFSET=165984

cryptsetup --cipher=serpent-xts-plain64 --hash=sha512 --key-size=512 --key-file=/dev/mapper/lukskey --keyfile-size=$KEY_SIZE --keyfile-offset=$OFFSET --align-payload 4096 --header header.img luksFormat $DEV_HDD
cryptsetup $--cipher=serpent-xts-plain64 --hash=sha512 --key-size=512 --key-file=/dev/mapper/lukskey --keyfile-size=$KEY_SIZE --keyfile-offset=$OFFSET --align-payload 4096 --header header.img luksFormat $DEV_SSD

# Open HDD and SSD
cryptsetup --header header.img --key-file=/dev/mapper/lukskey --keyfile-offset=$OFFSET --keyfile-size=$KEY_SIZE open $DEV_HDD enc_hdd
cryptsetup --header header.img --key-file=/dev/mapper/lukskey --keyfile-offset=$OFFSET --keyfile-size=$KEY_SIZE open $DEV_SSD enc_ssd
```

### Finish up encryption
```sh
cd /
cryptsetup close lukskey
umount /mnt
```

# LVM on LUKS

### Volume scheme
volume | device | size
------ | ------ | ----
swap | $SWAP_TARGET | 20G
system | ssd | 800G
data | hdd | 800G
FREE | ssd | 180G
FREE | hdd | 200G

### Create logic volumes
```sh
SWAP_TARGET=ssd # or 'hdd'

# vgcreate calls pvcreate automatically 
vgcreate ssd /dev/mapper/enc_ssd
vgcreate hdd /dev/mapper/enc_hdd

lvcreate -L 20G $SWAP_TARGET -n swap
lvcreate -L 800G ssd -n system
lvcreate -L 800G hdd -n data
```

# Partition formating

### Format devices
```sh
mkfs.btrfs /dev/ssd/system
mkfs.btrfs /dev/hdd/data
```

### Mount devices
```sh
mkdir /mnt/{system,data}
mount /dev/ssd/system /mnt/system
mount /dev/hdd/data /mnt/data
```

### Subvolume schemes
subvolume | mount point | volume
--------- | ----------- | ------
@ | / | system
@home | /home | system 
@data | /data | data
@data-$USER | @home/$USER/data | data

### Create all subvolumes
```sh
btrfs subvolume create /mnt/system/@
btrfs subvolume create /mnt/system/@home
btrfs subvolume create /mnt/data/@data
btrfs subvolume create /mnt/data/@data-$USER
```

### Finish up subvolume creation
```sh
umount /mnt/*
rm -rf /mnt/*
```

# Prepare filesystem
```sh
mount -o discard=async,noatime,compress=zstd,subvol=@ /dev/ssd/system /mnt
mkdir -p /mnt/{home,boot}

mount -o discard=async,noatime,compress=zstd,subvol=@home /dev/ssd/system /mnt/home
mount -o noatime,compress=zstd,subvol=@data /dev/hdd/data /mnt/data
```

### Create swap partition
```sh
mkswap /dev/ssd/swap
swapon /dev/ssd/swap
```

### Mount /boot
```sh
mount /dev/mapper/cryptboot /mnt/boot
mkdir /mnt/boot/efi

mount ${DEV_USB}1 /mnt/boot/efi
```

# Installation
### Refresh keys (optional)
Refreshing keys may take a while, be patient :)
```sh
pacman-key --refresh-keys
```

### Choosing fastest mirrors
```sh
pacman -Sy pacman-contrib

# Rank fastest mirrors from nearest locations
MIRRORLIST_URL="https://www.archlinux.org/mirrorlist/?protocol=https&use_mirror_status=on"
curl -s "$MIRRORLIST_URL&country=PL&country=DE&" | sed -e 's/^#Server/Server/' -e '/^#/d' | rankmirrors -n 5 - > /etc/pacman.d/mirrorlist

# Rank fastest mirrors in background while pacstraping
curl -s $MIRRORLIST_URL | sed -e 's/^#Server/Server/' -e '/^#/d' | rankmirrors -n 5 - > mirrorlist &
```

### Pacstrap
```sh
pacstrap /mnt base base-devel linux-hardened linux-firmware neovim networkmanager efibootmgr zsh lvm2 btrfs-progs
```

### Update mirrorlist
So the mirrorlist should probably be ranked by now. Check if the `mirrorlist` file exists with `cat mirrorlist` or wait till it is available.
```sh
mv mirrorlist /mnt/etc/pacman.d/mirrorlist
```


### Chroot and time zone
```sh
arch-chroot /mnt

ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
hwclock --systohc
```

### Localization
```sh
LANG=en_US.UTF-8

sed -i "s/^#$LANG/$LANG/" /etc/locale.gen
locale-gen

echo "LANG=$LANG" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
```

### Network
```sh
echo $HOSTNAME > /etc/hostname
echo "127.0.1.1	 $HOSTNAME.localdomain	 $HOSTNAME" > /etc/hosts
```

# Custom encrypt hook
### Download the hook
```sh

curl -sL https://git.io/JJ8IR | sed \
  -e "s/SIZE/$KEY_SIZE/" \
  -e "s/DEVICE_USB/$ID_USB/" \
  -e "s/DEVICE_HHD/$ID_HDD/" \
  -e "s/DEVICE_SSD/$ID_SSD/" \
  -e "s/OFFSET/$OFFSET/" > /etc/initcpio/hooks/waffencrypt
  
curl -sL https://git.io/JJ8Iu > /etc/initcpio/install/waffencrypt
```

### Add hook to the mkinitcpio
```sh
sed -i 's/^MODULES=()/MODULES=(loop ext2)/' /etc/mkinitcpio.conf
sed -i 's/modconf block/modconf block waffencrypt lvm2/' /etc/mkinitcpio.conf
```

# User setup

### Add user
```sh
useradd -m -G wheel -s $(which zsh) $USER
passwd $USER
```


# Generate fstab
### Main user data
Before generating fstab we have to mount the @data-$USER subvolume
```sh
mkdir /home/$USER/data
exit
mount -o noatime,compress=zstd,subvol=@data-$USER /dev/hdd/data /mnt/home/$USER/data
```

### Fstab
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

### Fixing genfstab stuff
Well, I've decided to use `-U` flag simply to have the UUID of `/boot` and `/boot/efi`. The rest of devices can use `/dev/mapper/<name>` as those won't change in future

Also genfstab creates `subvol=/@volume,subvol=@volume` so I removed one prepended with `/` from every btrfs entry

### Encrypted swap
```sh
nvim /etc/crypttab

# Add this entry at the end
swap /dev/mapper/ssd-swap /dev/urandom swap,cipher=twofish-xts-plain64,hash=sha512,size=512,nofail
```
```sh
nvim /etc/fstab

# Replace UUID in swap entry with
/dev/mapper/swap

# Set desired swapiness
echo vm.swappiness=10 > /etc/sysctl.d/99-swappiness.conf
```

### Back to archiso
```sh
arch-chroot /mnt
```

# visudo
Open up visudo with `EDITOR=nvim visudo` and add following at the top:

### Prevent editors which allow you to exit
```
Defaults env_reset
Defaults editor=/usr/bin/nvim, !env_editor
```

### Enable `wheel` group
Uncomment `%wheel ALL=(ALL) ALL`

# Disable root login
```sh
passwd -l root
```

# AUR helper
```sh
su -l $USER
git clone https://aur.archlinux.org/yay
cd yay
makepkg -si
```

# Secure boot
```sh
yay -S cryptboot sbupdate

# Exit the user account
exit

echo "cryptboot  /dev/disk/by-id/$UUID_USB  none  luks" > /etc/crypttab
cryptboot-efikeys create
cryptboot-efikeys enroll

cd /boot/efikeys

echo 'KEY_DIR="/boot/efikeys"' >> /etc/sbupdate.conf
echo 'ESP_DIR="/boot/efi"' >> /etc/sbupdate.conf
echo 'CMDLINE_DEFAULT="/vmlinuz-linux-hardened root=/dev/mapper/ssd-system rootflags=discard=async,compress=zstd,noatime,subvol=@ rw quiet"' >> /etc/sbupdate.conf
```

### mkinitcpio
```sh
mkinitcpio -p linux-hardened
sbupdate
```

### Boot entry
```sh
efibootmgr -c -d $DEV_USB -p 1 -L "Arch Linux Signed" -l "EFI\Arch\linux-hardened-signed.efi"

# Check if Arch Linux Hardened Signed is first in boot order
# If not set the correct boot order with
# efibootmgr -o XXXX,YYYY,ZZZZ
```

# Reboot
```sh
# Exit chroot
exit

umount -R /mnt
reboot
```

Enable secure boot in UEFI
