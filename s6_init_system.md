Do you want to start several processes inside your container ? Do you want to add logs to your services ? Do you want to be
able to handle unexpected termination of your processes and properly handle signals ? You probably want an init system in
your container.

## s6 in a container

Alpine uses the `openrc` init system.
You can add the `s6-overlay` package to your Alpine container. It is available on the edge repository.
```
sudo ~/alpine-make-rootfs --packages 's6-overlay' --timezone 'Europe/Paris' --branch 'edge' test.tar.gz
```

Start your container, and remove the `/sbin/init` symbolic link to `busybox`.
