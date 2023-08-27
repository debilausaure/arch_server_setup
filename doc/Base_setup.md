# Modern Arch Linux Install Guide 
--------

- Full Disk Encryption (LVM on LUKS, automatic passwordless decryption with TPM)
- SecureBoot (with Windows support)

--------

This guide is mostly a one-page synthesis of official [Installation Guide](https://wiki.archlinux.org/title/Installation_guide), where I summarize the steps of *my* usual Arch Linux setup.
Make sure you have a reasonably recent computer and an empty disk to install Linux on if you plan on following this guide.

## Preparing the bootable USB

ISO download : https://archlinux.org/download/

On Windows :
 - [Rufus](https://rufus.ie). Make sure to Select "UEFI" as target system as we'll need that.

On Linux :
```sh
dd if=arch.iso of=<path_to_your_usb_stick> bs=64K
```

## Live environment boot

Make sure to disable SecureBoot to boot the ISO and boot **in UEFI mode**.

## Keymap

Set the proper keymap :
```sh
loadkeys fr
```

## Verify boot mode

```sh
cat /sys/firmware/efi/fw_platform_size
```

If the file doesn't exist [you didn't boot in UEFI mode](#live-environment-boot).

## Connect to Internet

If you plugged an internet cable, internet should already be available.

WiFi :
```sh
iwctl
```
Get the name of your wifi card :
```sh
device list
```
Get your network SSID :
```sh
station <wifi-card> get-networks
```
Connect to the network and type the wifi password
```sh
station <wifi-card> connect <SSID>
```

Now verify connectivity :
```sh
ping archlinux.org
```

## Partition the disk

We will use the following disk layout :
 - A plain (non-encrypted) 500MiB FAT32 partition holding the bootable Unified Kernel Image (which will be authenticated by SecureBoot)
 - An LUKS encrypted partition on the remaining space that will contain the Linux filesystem.
   The encrypted partition is then subdivided in 3 "virtual" partitions with LVM :
    - A root partition holding the system
    - A home partition holding your personal files
    - A swap partition the size of your RAM to resume from
    - 256MiB of free space to be able to use `e2scrub`

First identify your empty disk :
```sh
fdisk -l
```

Then partition it :
```sh
sgdisk --clear \
       --new=1:0:+500MiB --typecode=1:ef00 \
       --new=2:0:0       --typecode=2:8300 <empty_disk>
```

Check the name of the partition on which you should setup encryption (type Linux Filesystem) :
```sh
fdisk -l <empty_disk>
```

## Fill the encrypted partition space with random data
```sh
cryptsetup open --type plain -d /dev/urandom /dev/<encrypted_partition> to_be_wiped
```
```sh
dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=1M
```
```sh
cryptsetup close to_be_wiped
```

## Create the encrypted partition
Go through the setup and choose a (preferably strong) password for your encrypted drive :
```sh
cryptsetup luksFormat /dev/<encrypted_partition>
```

Open the partition with your password :
```sh
cryptsetup open /dev/<encrypted_partition> encrypted_partition
```

## Create the sub-partitions
```sh
pvcreate /dev/mapper/encrypted_partition
```
```sh
vgcreate virtual_partitions /dev/mapper/encrypted_partition
```
Create the swap partition (replace `<ram_size>` with your RAM quantity :
```sh
lvcreate -L <ram_size>G virtual_partitions -n swap
```
Create the root partition (replace `<root_size>` with your desired size). I'd recommend going no lower than 64G :
```sh
lvcreate -L <root_size>G virtual_partitions -n root
```
Create the home partition, the partition will fill up all the remaining space :
```sh
lvcreate -l 100%FREE virtual_partitions -n home
```
Then we shave 256MiB off the home partition to be able to use `e2scrub` on the filesystems :
```sh
lvreduce -L -256M MyVolGroup/home
```

## Format the partitions
```sh
mkfs.ext4 /dev/virtual_partitions/root
```
```sh
mkfs.ext4 /dev/virtual_partitions/home
```
```sh
mkfs.fat -F32 /dev/<boot_partition>
```
```sh
mkswap /dev/virtual_partitions/swap
```

## Mount the partitions

```sh
mount /dev/virtual_partitions/root /mnt
```
```sh
mount --mkdir /dev/virtual_partitions/home /mnt/home
```
```sh
mount --mkdir /dev/<boot_partition> /mnt/efi
```
```sh
swapon --mkdir /dev/virtual_partitions/swap
```

## Setting up the initial filesystem
```sh
pacstrap -K /mnt base base-devel linux linux-firmware intel-ucode lvm2 efibootmgr man-db man
```

## Configure the initramfs hooks :
```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

Edit the new filesystem's `/etc/mkinitcpio.conf` :
```sh
vim /mnt/etc/mkinitcpio.conf
```
> ```
> ...  
> HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
> ...
> ```

## Configure the Unified Kernel Image

Find the UUID of the partition on which you created your password earlier (it has the type `part` and which contains `encrypted_partition` of type `crypt`) :
```sh
lsblk -o PATH,NAME,TYPE,UUID > /tmp/uuids
```

Edit the new filesystem's `/etc/kernel/cmdline` and `/etc/kernel/fallback_cmdline` :
```sh
vim /mnt/etc/kernel/cmdline
```
> ```
> rd.luks.name=<encrypted_partition_UUID>=encrypted_partition rd.luks.options=<encrypted_partition_UUID>=discard,tries=3 root=/dev/virtual_partitions/root resume=/dev/virtual_partitions/swap rw
> ```

Edit the new filesystem's `/etc/mkinitcpio.d/linux.preset` :
```sh
vim /mnt/etc/mkinitcpio.d/linux.preset
```
>```
># mkinitcpio preset file for the 'linux' package
>
>ALL_kver="/boot/vmlinuz-linux"`
>ALL_microcode=(/boot/*-ucode.img)`
>
>PRESETS=('default' 'fallback')
>
>default_uki="efi/EFI/Linux/arch-linux.efi"
>default_options="--splash=/usr/share/systemd/bootctl/splash-arch.bmp"
>
>fallback_uki="efi/EFI/Linux/arch-linux-fallback.efi"
>fallback_options="-S autodetect --cmdline /etc/kernel/fallback_cmdline"
>```

## Chroot into the new file system
```sh
arch-chroot /mnt
```

```sh
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
```

```sh
hwclock --systohc
```

```sh
nano /etc/locale-gen
```
>```
>...
>en_GB.UTF-8 UTF-8
>...
>```

```sh
locale-gen
```

```sh
nano /etc/locale.conf
```
>```
>LANG=en_GB.UTF-8
>```

```sh
nano /etc/vconsole.conf
```
>```
>KEYMAP=fr-latin1
>```

```sh
nano /etc/hostname
```
>```
><your_desired_hostname>
>```

```sh
passwd
```

## Generate the initramfs and Unified Kernel Image
```sh
mkinitcpio -P
```

## Add a boot entry for the UKI in UEFI
```sh
efibootmgr --create --disk /dev/<boot_partition> --part 1 --label "Arch Linux" --loader '\EFI\Linux\arch-linux.efi' --unicode
```

System should now be able to boot through a custom boot entry in your UEFI.

# Secure Boot

We deactivated Secure Boot to boot into the live Arch ISO, now we will configure it to allow our Linux Unified Kernel Image and reactivate it (and even retain the ability to boot Windows if that's what you want).

```sh
pacman -S sbctl
```
```sh
sbctl create-keys
```
```sh
sbctl enroll-keys -m #add -m if you'd like to dual boot windows
```
```sh
sbctl sign -s /efi/EFI/LINUX/arch-linux.efi
```
You only need to do this once: a pacman hook that signs the UKI whenever it gets updated was installed with `sbctl`.

## Retrieve the DBX

```sh
curl -OL https://uefi.org/sites/default/files/resources/x64_DBXUpdate.bin
```

Place the DBX update inside a FAT32 formatted usb key and boot into UEFI with the USB key plugged in. You should be able to set the DBX, and reactivate SecureBoot. 

TODO : TPM
