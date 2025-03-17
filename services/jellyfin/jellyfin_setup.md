Create a user and a group to run Jellyfin

```sh
sudo useradd -s /sbin/nologin -m -u 1003 -U jellyfin
```

Create a config and a cache directory
```sh
mkdir /home/jellyfin/cache
mkdir /home/jellyfin/config
chmod jellyfin:jellyfin /home/jellyfin/config/ /home/jellyfin/cache/
```
