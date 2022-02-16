# A list of additional steps to set up my server

## Install the microcode
```sh
pacman -S intel-ucode
```

## Set up boot through EFISTUB
```sh
pacman -S efibootmgr
```
```sh
blkid
```
```sh
efibootmgr --disk /dev/sdX --part X --create --label "Arch Linux" --loader /vmlinuz-linux --unicode 'root=PARTUUID=#rootPARTUUID# resume=PARTUUID=#swapPARTUUID# rw initrd=\intel-ucode.img initrd=\initramfs-linux.img' --verbose
```

## Enable network

Config a minimal systemd-networkd service.
First edit the link file:
```
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
```
vim /etc/systemd/network/20-wired.network
```
```
[Match]
Name=eno1

[Network]
DHCP=yes
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
useradd -m -G wheel -s /usr/bin/zsh debilausaure
```
```sh
passwd debilausaure
```
```sh
visudo
```
Uncomment the following line to allow users from group `wheel` to use sudo :  
`# %wheel ALL=(ALL:ALL) ALL`
