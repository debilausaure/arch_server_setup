## Containers

In this section we will be setting up containers for the server's services with `systemd-nspawn`. In particular, `systemd-nspawn` leverages namespaces to achieve isolation (including the file system hierarchy, the [process tree](https://www.man7.org/linux/man-pages/man7/pid_namespaces.7.html), the [IPC subsystems](https://www.man7.org/linux/man-pages/man7/ipc_namespaces.7.html) and the [host and domain name](https://www.man7.org/linux/man-pages/man7/uts_namespaces.7.html)). Additionally, [user namespacing](https://www.man7.org/linux/man-pages/man7/user_namespaces.7.html) and [network namespacing](https://www.man7.org/linux/man-pages/man7/network_namespaces.7.html) can be configured. 

### Manual container setup (educational purposes)

This part of the guide is meant to guide you through a "manual" toy container setup, in order to grasp the concepts manipulated by systemd when writing unit files. If you wish to skip to actual services setup through unit files, see the related page.

#### Getting the Alpine minimal rootfs

First, retrieve the alpine rootfs downloader script:
```sh
wget https://raw.githubusercontent.com/alpinelinux/alpine-make-rootfs/master/alpine-make-rootfs
chmod +x alpine-make-rootfs
```

Then, use the script to make a minimal alpine rootfs:
```sh
sudo ./alpine-make-rootfs --timezone 'Europe/Paris' alpine_minimal_rootfs-$(date +%Y%m%d).tar.gz
```
You can add packages to the rootfs by adding `--packages='<...>'` to the command line.

`cd` into the desired folder, and extract there the alpine rootfs:
```sh
mkdir somedir; cd somedir
tar -xzf alpine_minimal_rootfs-XXXX-XX-XX.tar.gz
```

You can now run Alpine in its own namespace. `-D` specifies the rootfs to use in the container, `-U` asks the kernel to use automatic [user namespaces](https://www.man7.org/linux/man-pages/man7/user_namespaces.7.html) (if your kernel supports it):
```sh
sudo systemd-nspawn -D somedir -U
```

Install the packages needed for your container, then I recommend that you mount the root filesystem as read-only.
```sh
sudo systemd-nspawn -D somedir --read-only -U
```

Depending on several configuration choices, you may have to add configuration options to the command-line:
- `--resolv-conf=` if you set up DNS with `systemd-resolved` as suggested in the Network part of the guide,  
use `--resolv-conf=bind-stub` to use the same domain name resolution as the host. You may need to remove the `/etc/resolv.conf` from the container to bind mount the file.

#### Persistent storage inside the container

You may want to allow persistent storage on the container, in this case I recommend you add a _system_ user to your host system, and mount its home directory inside the container:

```sh
sudo useradd -r -m -s /usr/bin/nologin container-user
```

When invoking the container, add the following command option `--bind-user=container-user` to bind the home folder of the host user inside the container.
Additionally, you may want to automatically set the working directory to that folder; in this case also add `--chmod=/run/host/home/container-user` to the command-line.

#### Placing the container inside a network namespace

If you just heard about network namespaces and would like to learn a bit more about it, I recommend you read [this pingnull's article](https://pingnull.com/linux-networking-namespaces/). It guides you through creating your own network namespace and setting up network access.

I recommend that you place the container inside a network namespace. According to [`man 7 network_namespaces`](https://www.man7.org/linux/man-pages/man7/network_namespaces.7.html), a network namespace isolates the system resources associated with networking, which is not done by default when starting a container through the `systemd-nspawn` command line.

In particular:
- If your container does not need to perform any operations on the network, I recommend that you use the option `--private-network`, which will completely isolate your container network-wise. The container only has access to a loopback interface, isolated from the host's loopback.
- If your container does need access to the network, I recommend that you use the option `--network-zone=<...>`. This will create a virtual interface inside your container, linked to an automatically created network bridge inside your host. All the containers spawned with the same `network-zone` name will be linked to the same bridge.
  On the container, you will need to set the interface to up and get a dhcp lease from the host, along with a DNS and a default route. In Alpine Linux :
  ```
  ip link set dev host0 up
  udhcpc -i host0
  ```
  **Note :** By default, `systemd-networkd` sets up a NAT; your host is used as a proxy for your container's traffic. Now you probably want to filter some of the traffic coming from/headed to your container, otherwise it kind of defeats the whole point of isolating your container's network. Unfortunately, this cannot be done through `systemd`.

#### Firewalling the container with `iptables`
