# Setting up OpenVPN Server on CoreOS

Due to issues with getting wireguard working reverting to OpenVPN via [kylemanna/docker-openvpn](Using https://github.com/kylemanna/docker-openvpn).

I'm going to assume a domain of ghost.algebraic.ninja

First, create the data volume.
```
OVPN_DATA="ovpn-data-ghost"
docker volume create --name $OVPN_DATA
```

Now show the details for reference with `docker volume inspect $OVPN_DATA`

??? "example output"
    [
        {
            "CreatedAt": "2019-05-22T20:00:38Z",
            "Driver": "local",
            "Labels": {},
            "Mountpoint": "/var/lib/docker/volumes/ovpn-data-ghost/_data",
            "Name": "ovpn-data-ghost",
            "Options": {},
            "Scope": "local"
        }
    ]

Next we'll initialise the `$OVPN_DATA` container that will hold the configuration files and certificates. The container will prompt for a passphrase to protect the private key used by the newly generated certificate authority.

```
VPN_HOST=ghost.algebraic.ninja
docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm kylemanna/openvpn ovpn_genconfig -u udp://$VPN_HOST
```
Make a note of the output from the config
??? info "config"
    ``` hl_lines="3"
    Status: Downloaded newer image for kylemanna/openvpn:latest
    Processing PUSH Config: 'block-outside-dns'
    Processing Route Config: '192.168.254.0/24'
    Processing PUSH Config: 'dhcp-option DNS 8.8.8.8'
    Processing PUSH Config: 'dhcp-option DNS 8.8.4.4'
    Processing PUSH Config: 'comp-lzo no'
    Successfully generated config
    Cleaning up before Exit ...
    ```
docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm -it kylemanna/openvpn ovpn_initpki
```

Start OpenVPN server process

```
docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
```

Generate a client certificate without a passphrase

```
CLIENTNAME=jamesveitch
docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm -it kylemanna/openvpn easyrsa build-client-full $CLIENTNAME nopass
```

Retrieve the client configuration with embedded certificates

```
docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm kylemanna/openvpn ovpn_getclient $CLIENTNAME > $CLIENTNAME.ovpn
```

Now add it as a `systemd` service on reboot. Download the docker-openvpn@.service file to /etc/systemd/system:
```
curl -L https://raw.githubusercontent.com/kylemanna/docker-openvpn/master/init/docker-openvpn%40.service | sudo tee /etc/systemd/system/docker-openvpn@.service
```
Enable and start the service with:
```
systemctl enable --now docker-openvpn@ghost.service
```