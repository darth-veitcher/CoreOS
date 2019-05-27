# Setting up rclone for backups and mounts
Using rclone on CoreOS is a little more involved due to some of the missing dependencies, lack of a package manager and a read-only filesystem which is designed to prevent permanent modifications.

Luckily, CoreOS allows you to use `/opt/bin` and everything in here is considered part of `$PATH`.

```bash
sudo mkdir -p /opt/bin
```

## FUSE
In order to use `rclone mount ..` we need to have fuse available on CoreOS. Obviously it would be too straightforward for it to already be here so we'll need to use a docker container to compile the `fusermount` binary and then copy it to `/opt/bin`.

```bash
# Run our container, copy the binary required and then remove it
sudo docker run -d --name fuse saracen9/fuse-builder sleep && \
    sudo docker cp fuse:/usr/local/bin/fusermount3 /opt/bin/fusermount && \
    docker rm -f fuse
```

## Rclone
We can download and extract rclone directly to here to make life easy.

```bash
sudo wget "https://downloads.rclone.org/rclone-current-linux-amd64.zip" && \
    sudo unzip rclone* && sudo rm -f rclone*.zip && \
    sudo mv rclone* rclone && sudo mv rclone/rclone /opt/bin && \
    sudo chmod +x /opt/bin/rclone
```

Now create your `~/.rclone.conf` according to instructions elsewhere and you can run a command similar to the below:

```bash
rclone mount google: /mnt/gdrive --daemon --allow-other --drive-chunk-size 32M --timeout 1h --umask 002
```