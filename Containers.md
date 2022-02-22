## Containers

In this section we will be setting up containers for the server's services with `systemd-nspawn`.

### Getting the Alpine minimal rootfs

First, retrieve the alpine rootfs downloader script:
```sh
wget https://raw.githubusercontent.com/alpinelinux/alpine-make-rootfs/master/alpine-make-rootfs
chmod +x alpine-make-rootfs
```

Then, use the script to make a minimal alpine rootfs:
```sh
sudo ./alpine-make-rootfs --branch v3.15 --timezone 'Europe/Paris' alpine3.15-$(date +%Y%m%d).tar.gz
```

