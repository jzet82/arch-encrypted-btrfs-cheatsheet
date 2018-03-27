# Install Arch Linux on an Encrypted Btrfs Partition

**This guide is not suitable for uefi mode**

**Keyboard layout**
```bash
loadkeys pl
```

**Find your harddisk device name**
```bash
cat /proc/partitions
```

**Overwrite it with some random data**
```bash
badblocks -c 10240 -s -w -t random -v /dev/sda
```

**Create two partitions**
```
/dev/sda1 200M for /boot
/dev/sda2 the remaining space for the system
```

**Format the boot partition with ext2 or ext4**
```bash
mkfs.ext4 /dev/sda1
```

**Create cryptographic device mapper device in LUKS encryption mode**
```bash
cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 256 --hash sha256 --use-random /dev/sda2
```

**Unlock the partition**
```bash
cryptsetup luksOpen /dev/sda2 cryptroot
```

**Format the root volume with btrfs**
```bash
mkfs.btrfs -L Arch /dev/mapper/cryptroot
mount -o noatime,compress=lzo,discard,ssd,defaults /dev/mapper/cryptroot /mnt
```

**Create Btrfs subvolumes**
```bash
btrfs subvolume create /mnt/__active
btrfs subvolume create /mnt/__active/rootvol
btrfs subvolume create /mnt/__active/home
btrfs subvolume create /mnt/__active/var
btrfs subvolume create /mnt/__snapshots
```

**Mount subvolumes**
```bash
umount /mnt
mount -o noatime,compress=lzo,discard,ssd,defaults,subvol=__active/rootvol /dev/mapper/cryptroot /mnt
mkdir /mnt/{home,var}
mount -o noatime,compress=lzo,discard,ssd,defaults,subvol=__active/home /dev/mapper/cryptroot /mnt/home
mount -o noatime,compress=lzo,discard,ssd,defaults,subvol=__active/var /dev/mapper/cryptroot /mnt/var
```

**Mount boot partition**
```bash
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

**Bootstrap the system**
```bash
pacstrap /mnt base base-devel btrfs-progs
```

**Generate /etc/fstab**
```bash
genfstab -p /mnt >> /mnt/etc/fstab
```

**Chroot into the new system**
```bash
arch-chroot /mnt
```

**Create boot image**

`
Modify /etc/mkinitcpio.conf by adding encrypt before filesystems inside the HOOKS section and run
`
```bash
mkinitcpio -p linux
```

**Set the timezone**
```bash
ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
```

**Generate the required locales**

`
Uncomment the en_US.UTF-8 and pl_PL.UTF-8 languages in /etc/locale.gen and run
`
```bash
locale-gen
```

**In /etc/locale.conf, write**
```bash
LANG=pl_PL.UTF-8
LC_COLLATE=C
```

**Set the hardware clock to UTC**
```bash
hwclock --systohc --utc
```

**Set the desired hostname**
```
echo Arch Desktop > /etc/hostname
```

**Set the root password**
```bash
passwd
```

**Install Grub**
```bash
pacman -Sy grub
```

**Install few other important packages**
```bash
pacman -Sy linux-headers wpa_supplicant wireless_tools
```

**Add the following kernel parameter to be able to unlock your LUKS encrypted root partition during system startup in `/etc/default/grub`**
```bash
GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda2:cryptroot"
```

**Generate grub.cfg**
```bash
grub-install --recheck /dev/sda
grub-mkconfig --output /boot/grub/grub.cfg
```

**Exit from chroot, unmount the partitions, close the device and reboot**
```
exit
umount -R /mnt/boot
umount -R /mnt
cryptsetup close cryptroot
reboot
```
