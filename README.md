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
> `# %wheel ALL=(ALL:ALL) ALL`

## Enable network 
