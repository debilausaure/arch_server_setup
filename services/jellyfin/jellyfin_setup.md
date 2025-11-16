Create a user and a group to run Jellyfin

```sh
sudo useradd -s /sbin/nologin -m -u 1003 -U jellyfin
```

If not installed already, install the Nvidia driver:
```
yay -S nvidia
```
then reboot.

Install the Nvidia container tools:
```
yay -S nvidia-container-toolkit
```

Configure Docker to make it use the Nvidia Container Runtime:
```
sudo nvidia-ctk runtime configure --runtime=docker
```
and restart docker:
```
sudo systemctl restart docker
```

Create a config and a cache directory
```sh
mkdir /home/jellyfin/cache
mkdir /home/jellyfin/config
chmod jellyfin:jellyfin /home/jellyfin/config/ /home/jellyfin/cache/
```
