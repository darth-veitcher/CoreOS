# Enable firewall
After ensuring that the VPN works we're going to disable all access to ssh or any other protocol externally except for http 80 / https 443.

That way we can subsequently use Traefik as a reverse proxy into the VPN.

For now, let's simply block direct access to SSH.

Check the status of the firewall
```bash
systemctl status -l iptables-restore
```

??? "output"
    ```
    core@ghost ~ $ systemctl status -l iptables-restore
    ● iptables-restore.service - Restore iptables firewall rules
    Loaded: loaded (/usr/lib/systemd/system/iptables-restore.service; disabled; vendor preset: disabled)
    Active: inactive (dead)
    ```

So it's not enabled, let's enable it.
```bash
sudo systemctl enable iptables-restore
```

We'll now set some rules in `/var/lib/iptables/rules-save`

??? info "/var/lib/iptables/rules-save"
    ```
    *filter
    :INPUT DROP [0:0]
    :FORWARD DROP [0:0]
    :OUTPUT ACCEPT [0:0]

    # Accept all loopback (local) traffic:
    -A INPUT -i lo -j ACCEPT

    # Keep existing connections (like our SSH session) alive:
    -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

    # Accept all TCP/IP traffic to HTTP, and HTTPS ports
    -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
    -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT

    # Accept all UDP to our VPN
    -A INPUT -p udp --dport 1194 -j ACCEPT

    # (Temporary)
    # Accept all TCP/IP to SSH
    -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT

    # Accept pings:
    -A INPUT -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A INPUT -p icmp -m icmp --icmp-type 3 -j ACCEPT
    -A INPUT -p icmp -m icmp --icmp-type 11 -j ACCEPT
    COMMIT
    ```

This config:

* By default:
    * Drops all incoming connection attempts
    * Prevents any forwarding attempts
    * Allows outbound traffic
* Overrides:
    * Allows ICMP pings and echo responses
    * Allows inbound to http (80) and https (443)
    * Allows inbound udp to our VPN
* For the moment:
    * Allows inbound to port 22 for ssh

Set the right permissions

```bash
sudo chmod 0644 /var/lib/iptables/rules-save
```

Now we should be ready to try the service again:
```bash
sudo systemctl start iptables-restore
```

If successful, `systemctl` will exit silently. We can check the status of the firewall in two ways. First, by using `systemctl` status:
```bash
sudo systemctl status -l iptables-restore
```
And secondly by listing the current iptables rules themselves:
```bash
sudo iptables -v -L
```

??? "current rules example output"
    ```
    Chain INPUT (policy DROP 0 packets, 0 bytes)
    pkts bytes target     prot opt in     out     source               destination         
        4   200 ACCEPT     all  --  lo     any     anywhere             anywhere            
    69 30530 ACCEPT     all  --  any    any     anywhere             anywhere             ctstate RELATED,ESTABLISHED
        0     0 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:http
        0     0 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:https
        0     0 ACCEPT     udp  --  any    any     anywhere             anywhere             udp dpt:openvpn
        1    64 ACCEPT     tcp  --  any    any     anywhere             anywhere             tcp dpt:ssh
        0     0 ACCEPT     icmp --  any    any     anywhere             anywhere             icmp echo-reply
        0     0 ACCEPT     icmp --  any    any     anywhere             anywhere             icmp destination-unreachable
        0     0 ACCEPT     icmp --  any    any     anywhere             anywhere             icmp time-exceeded
    ```

All clients on the VPN will be allocated to the `192.168.255.0/24` subnet.

## Hardening the firewall
With the above setup and confirmed to work we'll now remove direct external access to SSH. We will though allow from the VPN subnet and localhost. This requires editing the `/var/lib/iptables/rules-save` file again.

### Understand local network details
Before modifying the firewall we should understand the current network topology. If we want to limit access to `host` resources (i.e. CoreOS) then we need to appropriately filter network traffic from our docker containers. Some further details available from this StackOverflow question: [From inside of a Docker container, how do I connect to the localhost of the machine?](https://stackoverflow.com/questions/24319662/from-inside-of-a-docker-container-how-do-i-connect-to-the-localhost-of-the-mach#24326540)

Running `sudo ip addr show docker0` will output something similar to the following.
???+ info "host network"
    ```
    core@ns3007580 ~ $ sudo ip addr show docker0
    3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
        link/ether 02:42:90:8f:d9:1f brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
            valid_lft forever preferred_lft forever
        inet6 fe80::42:90ff:fe8f:d91f/64 scope link 
            valid_lft forever preferred_lft forever
    ```
As you'll see from the above the subnet for docker on the host is `172.17.0.1/16` and our host will therefore be `172.17.0.1`. Once connected to the VPN with a client you should be able to connect with a `ssh core@172.17.0.1` command. **Check this first before proceeding**.

### Modify the firewall
`sudo vi /var/lib/iptables/rules-save`

The below diff shows the changes to make
```diff
- # (Temporary)
- # Accept all TCP/IP to SSH
- -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
+ # Accept all TCP/IP to SSH from VPN subnet
+ -A INPUT -p tcp -m tcp --dport 22 --src 172.17.0.1/16 -j ACCEPT
```

???+ warning
    The next step could permanently lock you out of the system if the previous steps have not been performed correctly...

    Fair warning!

Now reboot and apply the changes made. You shouldn't be able to ssh directly into the box now from an external IP address.
```bash
sudo shutdown -r now
```

SSH from a connected VPN client should work though.
```bash
ssh core@172.17.0.1
```