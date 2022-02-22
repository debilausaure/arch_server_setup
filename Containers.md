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
sudo ./alpine-make-rootfs --timezone --packages 'apk-tools' 'Europe/Paris' alpine_minimal_rootfs-$(date +%Y%m%d).tar.gz
```

`cd` into the desired folder, and extract there the alpine rootfs:
```sh
mkdir somedir; cd somedir
tar -xzf alpine_minimal_rootfs-XXXX-XX-XX.tar.gz
```

You can now run Alpine in its own namespace. `-D` specifies the rootfs to use in the container, `-U` asks the kernel to use automatic namespaces:
```sh
sudo systemd-nspawn -D somedir -U
```

Install the packages needed for your container, then I recommend that you mount the root filesystem as read-only.
```sh
sudo systemd-nspawn -D somedir --read-only -U
```

Depending on several configuration choices, you may have to add configuration options to the command-line:
- `--resolv-conf=` if you set up DNS with `systemd-resolved` as suggested in the Network part of the guide,  
use `--resolv-conf=bind-stub` to use the same domain name resolution as the host.

### Persistent storage inside the container

You may want to allow persistent storage on the container, in this case I recommend you add a _system_ user to your host system, and mount its home directory inside the container:

```sh
sudo useradd -r -m -s /usr/bin/nologin container-user
```

When invoking the container, add the following command option `--bind-user=container-user` to bind the home folder of the host user inside the container.
Additionally, you may want to automatically set the working directory to that folder; in this case also add `--chmod=/run/host/home/container-user` to the command-line.
