# A list of additional steps to set up my server

## Install the microcode patches
```sh
pacman -S intel-ucode
```

## Enabling [TRIM](https://wiki.archlinux.org/title/Solid_state_drive#TRIM)

TRIM allows for wear-leveling on SSD, increasing the lifespan of your disk. The technology is only relevant on flash-based storage.
To enable continuous TRIM, edit `/etc/fstab`:
```sh
vim /etc/fstab
```
Add the `discard` mount option to your partitions. The option is notably available on `FAT`, `ext4` and `swap` partitions.
```
...
UUID=XXXXXXXX-XXXX-XXXX-XXXXXXXXXXXX  /           ext4 rw,relatime,discard  0 1
...
```

**Note :** if you use a relatively old machine, make sure that it supports at least SATA rev3.1 (~mid 2011). In that case continuous TRIM may severely degrade your performances. In that case, prefer using periodic TRIM.


## Boot directly from the UEFI firmware with EFISTUB
```sh
pacman -S efibootmgr
```
```sh
blkid
```
```sh
efibootmgr --disk /dev/sdX --part X --create --label "Arch Linux" --loader /vmlinuz-linux --unicode 'root=PARTUUID=#rootPARTUUID# resume=PARTUUID=#swapPARTUUID# rw initrd=\intel-ucode.img initrd=\initramfs-linux.img' --verbose
```

## Network setup

Config a minimal `systemd-networkd` service.
First edit the link file:
```sh
vim /etc/systemd/network/20-wired.link
```
```
[Match]
MACAddress=xx:xx:xx:xx:xx:xx

[Link]
NamePolicy=kernel data onboard slot path
MACAddressPolicy=persistent
```
If you want to set up Wake On Lan, add the following line to the file:
```
WakeOnLan=magic
```

Then, edit the network file:
```sh
vim /etc/systemd/network/20-wired.network
```
```
[Match]
Name=eno1

[Network]
DHCP=yes
```

Finally, enable the `systemd-networkd` service.
```sh
systemctl enable systemd-networkd.service
```

### Domain name resolution

To set up name resolution, enable the `systemd-resolved` service.
```sh
systemctl enable systemd-resolved.service
```

To stay compatible with applications that read from `/etc/resolv.conf` directly, replace `/etc/resolv.conf` with a symlink to a config file listing the local DNS stub provided by `systemd-resolved`:
```sh
ln -rsf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

## Set up a ssh server for remote connection
Install `openssh`:
```sh
pacman -S openssh
```

I **strongly** recommend that you harden you ssh server configuration. For example, see the [Mozilla hardening recommendations](https://infosec.mozilla.org/guidelines/openssh).  
Additionnally, I recommend that you explicitly configure which users are allowed to login through ssh. Edit `/etc/ssh/sshd_config`:
```sh
...
AllowUsers  <user>
...
```

## Install zsh and the grml config
```sh
pacman -S zsh grml-zsh-config
```

## Set up a new user with sudo access and zsh shell
```sh
pacman -S sudo
```
```sh
useradd -m -G wheel -s /usr/bin/zsh <user>
```
```sh
passwd <user>
```
```sh
visudo
```
Uncomment the following line to allow users from group `wheel` to use sudo :  
`# %wheel ALL=(ALL:ALL) ALL`

## 
