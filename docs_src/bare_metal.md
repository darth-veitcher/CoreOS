# Installing CoreOS to a Bare Metal host
If, like, me you want to run this on-prem then this chapter will quickly run through how to get setup in an environment where you've got a physical host and don't want to rely on a cloud provider. We're going to download an iso, install to disk and then run through some configuration specifics for my environment. Bits might be useful / relevant to you depending on your own setup so YMMV.

**Obtain the latest image**
Navigate to the [CoreOS, Container Linux](https://coreos.com/os/docs/latest/booting-with-iso.html) website and download the latest iso file.

```bash
cd ~/Downloads
wget https://stable.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso
```

With the image downloaded now write it to disk with a `sudo dd if=~/Downloads/coreos_production_iso_image.iso of=/dev/path/tousb bs=1m` same as any other iso.

**Boot the host using usb**
Couple of things to note:

1. CoreOS relies on BIOS as opposed to UEFI
2. Make sure you set the USB emulation to Hard Disk if given the option in your BIOS

Once booted up you can run a `sudo passwd core` to set a password on the main user and then head back to a screen somewhere else to continue the setup over ssh.

## Configuration
There's a few examples on the [official website](https://coreos.com/os/docs/latest/clc-examples.html) but the best documentation currently resides in the [GitHub repo](https://github.com/coreos/container-linux-config-transpiler/blob/master/doc/configuration.md).

My particular setup at home has the following characteristics:

**Network bonding**

* Physical rack-mounted hosts each with +4 NICs (Dell R610 and Dell 710)
* Managed switch with configured LACP for bonded network interfaces

As a result I'll also be configuring the CoreOS images to run bonded network 802.3ad NICs to maximise network throughput.

**Fileshares**

I have a seperate physcial fileserver (called `Lake`) and so will be mounting NFS/CIFS shares into the CoreOS system so that they can be accessed by docker images.

**Local Filesystem**

On each of the servers I've got a slightly different storage setup. I'll mount these extra disks (over and above the standard root) into the host under the `/data` directory.

**Avahi Network Discovery**

For the sake of ease I'm going to run avahi to broadcast/multicast my server hostnames etc. on the local network. There's a custom docker image, associated systemd service and a config file being created as a result.

### Config vs Ignitiion
Container Linux has the concept of creating a human-readable `Container Linux Config` and then compiling this into a condensed (and validated) machine-readable format called `Ignition`. As a result we'll create a config file and then `transpile` it into an ignition one (which is actually what gets loaded and read by CoreOS). The tool for performing this can be found on GitHub [here](https://github.com/coreos/container-linux-config-transpiler).

#### Configuratiuon
My configs can be found [here]


Either ssh into the machine and run `vi config` to copy/paste the above config file in or `scp` it across and then ssh in. Now we need to get the config transpiler. Go to the [releases](https://github.com/coreos/container-linux-config-transpiler/releases) to find the latest one.

##### Transpile the config
```bash
wget https://github.com/coreos/container-linux-config-transpiler/releases/download/v0.9.0/ct-v0.9.0-x86_64-unknown-linux-gnu && \
    sudo mkdir -p /opt/bin && \
    sudo mv ct* /opt/bin/ct && \
    sudo chmod +x /opt/bin/ct
```

Typing `ct --help` should now return the following output in the terminal.

??? info "ct"
    ```
    Usage of ct:
    -files-dir string
            Directory to read local files from.
    -help
            Print help and exit.
    -in-file string
            Path to the container linux config. Standard input unless specified otherwise.
    -out-file string
            Path to the resulting Ignition config. Standard output unless specified otherwise.
    -platform string
            Platform to target. Accepted values: [azure digitalocean ec2 gce packet openstack-metadata vagrant-virtualbox cloudstack-configdrive custom].
    -pretty
            Indent the resulting Ignition config.
    -strict
            Fail if any warnings are encountered.
    -version
            Print the version and exit.
    ```

We'll now transpile the config.

```bash
ct -platform custom -in-file ~/config -out-file ~/ignition.json
```

If there aren't any errors you can review the minified output file.
??? "cat ~/ignition.json"
    ```
    {"ignition":{"config":{},"security":{"tls":{}},"timeouts":{},"version":"2.2.0"},"networkd":{"units":[{"contents":"[Match]\nName=bond1\n\n[Network]\nDHCP=ipv4\n","name":"30-bond1.network"},{"contents":"[NetDev]\nName=bond1\nKind=bond\n\n[Bond]\nMode=802.3ad\nMIIMonitorSec=1s\nLACPTransmitRate=fast\nUpDelaySec=2s\nDownDelaySec=8s\n","name":"30-bond1.netdev"},{"contents":"[Match]\nMACAddress=D4:BE:D9:A9:B5:54 D4:BE:D9:A9:B5:56 D4:BE:D9:A9:B5:58 D4:BE:D9:A9:B5:5A\n\n[Network]\nBond=bond1\n","name":"30-bond1-slaves.network"}]},"passwd":{"users":[{"name":"core","sshAuthorizedKeys":["ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDZsCEU/2uFV4YpBjoOsSIY1yRta3mGdd81TyvZFGzVsXEn7BbkJXPI6I3r8vXQaRgQvr//yj/Q3whuGlcBuH8PuCAUlHg2oJIMJ+NsIW/E300nzu0j8lltDvLg4Sl1Ncag4Hy5JtjeWyoouHCUajxN8jRKqXW1pS3hZO2+UCN2t6ZNl7n01cZZviVWcoPe2tUpy2O52iWW6Wt7cgWFSBVPCmnD3p5Vwnz2d5SrSgzQ+9Qq/jrU0ZGF32wLt/c3OzHMBRLYNJviMaEZfonIjTmpqOxUQxXzO25K3/A0QHeEtBInKpNr7TUJ/U0rtpNrw2Th6wsc4pjLLM9R9U6EbH9D james@jamesveitch.com"]}]},"storage":{"files":[{"filesystem":"root","path":"/etc/hostname","contents":{"source":"data:,asimov","verification":{}},"mode":420},{"filesystem":"root","path":"/etc/coreos/update.conf","contents":{"source":"data:,GROUP%3Dstable%0AREBOOT_STRATEGY%3D%22reboot%22%0ALOCKSMITHD_REBOOT_WINDOW_START%3D%22Thu%2004%3A00%22%0ALOCKSMITHD_REBOOT_WINDOW_LENGTH%3D%222h%22","verification":{}},"mode":420}],"filesystems":[{"mount":{"device":"/dev/disk/by-uuid/0192c047-0792-4f7a-9f27-f093dd79bd2e","format":"btrfs","label":"DATA","wipeFilesystem":true},"name":"data"}]},"systemd":{"units":[{"contents":"[Unit]\nDescription=Automount Lake Network Share (TV)\n\n[Automount]\nWhere=/mnt/scratch\n\n[Install]\nWantedBy=multi-user.target\n","enabled":true,"name":"mnt-lake-tv.automount"},{"contents":"[Unit]\nDescription=Lake Network Share (TV)\n\n[Mount]\nWhat=lake.local:/export/TV\nWhere=/mnt/lake/tv\nType=nfs\n\n[Install]\nWantedBy=multi-user.target\n","enabled":true,"name":"mnt-lake-tv.mount"},{"contents":"[Unit]\nDescription=Automount Lake Network Share (Movies)\n\n[Automount]\nWhere=/mnt/scratch\n\n[Install]\nWantedBy=multi-user.target\n","enabled":true,"name":"mnt-lake-movies.automount"},{"contents":"[Unit]\nDescription=Lake Network Share (Movies)\n\n[Mount]\nWhat=lake.local:/export/Movies\nWhere=/mnt/lake/movies\nType=nfs\n\n[Install]\nWantedBy=multi-user.target\n","enabled":true,"name":"mnt-lake-movies.mount"}]}}
    ```

## Installation
With the ignition file now created it's time to install...

```bash
sudo coreos-install -d /dev/sda -i ignition.json

# Once completed reboot
sudo shutdown -r now
```
