# Arch linux encrypted install
## Disk preperation of `/dev/<drive_to_partition>`

```
fdisk -l
cfdisk /dev/<drive_to_partition>
```

Create one boot partition of at least 550M (efi mode), one swap partition for 'suspend to disk' of at least 2G and a root partition of the remaining space.
If you want, you can create a seperate home partition, that can be encrypted.

### Format Disks
```
mkfs.fat -F32 /dev/<boot_partition>

cryptsetup luksFormat /dev/<root_partition>
cryptsetup open /dev/<root_partition> <crypt_root_name>
mkfs.ext4 /dev/mapper/<crypt_root_name> or mkfs.ext4 /dev/<root_partition>

(Seperate home partition)
cryptsetup luksFormat /dev/<home_partition>
cryptsetup open /dev/<home_partition> <crypt_home_name>
mkfs.ext4 /dev/mapper/<crypt_home_name> or mkfs.ext4 /dev/<home_partition>

(Encrypted swap partition)
cryptsetup luksFormat /dev/<swap_partition>
cryptsetup open /dev/<swap_partition> <crypt_swap_name>

mkswap /dev/mapper/<crypt_swap_name> or mkswap /dev/<swap_partition>
swapon /dev/mapper/<crypt_swap_name> or swapon /dev/<swap_partition>
```

### Mount disks
```
mount /dev/mapper/<crypt_root_name> /mnt or mount /dev/<root_partition>

mkdir /mnt/boot
mount /dev/<boot_partition> /mnt/boot

(Seperate home partition)
mkdir /mnt/home
mount /dev/mapper/<crypt_home_name> /mnt/home or mount /dev/<home_partition> /mnt/home
```

## Install the operating system
`pacstrap /mnt base base-devel linux linux-firmware amd-ucode vim man`
`genfstab -U /mnt >> /mnt/etc/fstab`

## Chroot into the installed system
`arch-chroot /mnt`

## Early system preperation
Uncomment `en_US.UTF-8` in `/etc/locale.gen`
`locale-gen`
Edit `/etc/locale.conf` and add `LANG=en_US.UTF-8`

Edit `/etc/vconsole.conf` and add `KEYMAP=<keyboard layout>`

Edit `/etc/hostname` and add `<hostname>`

Edit `/etc/hosts` and add:
```
127.0.0.1	localhost
::1			localhost
127.0.1.1	<hostname>.<localdomain>	<hostname>
```

## Encryption configuration
Edit `/etc/mkinitcpio.conf` and change `HOOKS=(...)` to `HOOKS=(... modconf keyboard keymap ... encrypt resume filesystems`

Only add `resume` if you plan on encrypting the swap partition.

`mkinitcpio -p linux`

Edit `/etc/crypttab` and add:
```
...
<crypt_home_name>	UUID=<UUID_crypt_home>
...
```

(Encrypted swap)

Edit `/etc/fstab` and add:
```
# /dev/mapper/<crypt_swap_name>
/dev/mapper/<crypt_swap_name>	swap	swap	defaults 0 0
```

## Install grub bootloader
`pacman -S grub efibootmgr dosfstools mtools`

Edit `/etc/default/grub` and change:
```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=<crypt_root_uuid>:<crypt_root_name> root=/dev/mapper/<crypt_root_name> resume=/dev/mapper/<crypt_swap_name>"
```
Only add resume if you are encrypting the swap partition.

`grub-install --efi-directory=/boot`µ

`grub-mkconfig -o /boot/grub/grub.cfg`

## Enable networking
`pacman -S networkmanager`

`systemctl enable NetworkManager`

## Exit the system and ENJOY
```
exit
umount -ar
reboot
```
