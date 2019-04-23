---
title: Configuring Proxmox with one IPv4 on Kimsufi
date: 2019-03-27 12:09:33
tags:
  - Kimsufi
  - Proxmox
  - NAT
categories:
  - cheatsheets
author: Jakub Papuga
---
## Topology cheatsheet

|  | Internal IP | Forwarded Ports |
|:---|:---:|:---:|:---:|
| Proxmox Node | 192.168.0.254 | --- |
| NGINX Reverse Proxy | 192.168.0.100 | TCP: 80 & 443 |
| Plex Media Server| 192.168.0.101 | TCP: 3005; 32400; 8324; 32469 UDP: 1900; 5353; 32410 |

## Configuring new interface

Start by adding the following to the `/etc/network/interfaces` file:

#### /etc/network/interfaces

```
auto vmbr2
iface vmbr2 inet static
    address 192.168.0.254
    netmask 255.255.255.0
    bridge_ports none
    bridge_stp off
    bridge_fd 0
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up iptables -t nat -A POSTROUTING -s '192.168.0.0/24' -o vmbr0 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '192.168.0.0/24' -o vmbr0 -j MASQUERADE
```

and bring the `vmbr2` interface up by executing:

```
ifup vmbr2
```

You can now create containers that will be able to connect to the outside world, but not the other way around. For incoming connections to work, you will need to forward a few ports. Also, make sure to choose `vmbr2` while creating the container!

![Make sure to choose "vmbr2"](https://i.mrpsycho.pl/selif/b7phh7lc.png)

You can't SSH into the container right now as no port is forwarded to it, but you can enter the container by using the Proxmox built-in console (if you've chosen to use password authentication) or by executing `lxc-attach 100` on the node to attach to the container directly, bypassing the SSH authentication. From there you can run `ping 8.8.8.8` to check if the container has an internet connection. It may also be a good opportunity to adjust your SSH config if you're using password authentication as some LXC templates do not allow root login with a password.

```
root@nginxProxy:~# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=122 time=13.5 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=122 time=13.5 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=122 time=13.5 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 13.524/13.571/13.598/0.033 ms
```

## Port Forwarding

Now we can start forwarding ports to your containers.

I suggest to [randomize](https://numbergenerator.org/randomnumbergenerator/1024-65535) the SSH port and check if it's not a [well-known](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers#Well-known_ports) port.
The general procedure is to add an iptables command to post-up and post-down of the interface:

#### /etc/network/interfaces

```
...
post-up iptables -t nat -A PREROUTING -i [PUBLIC_INTERAFCE] -p [TCP|UDP] --dport [EXTERNAL_PORT] -j DNAT --to [INTERNAL_IP]:[INTERNAL_PORT]
post-down iptables -t nat -D PREROUTING -i [PUBLIC_INTERAFCE] -p [TCP|UDP] --dport [EXTERNAL_PORT] -j DNAT --to [INTERNAL_IP]:[INTERNAL_PORT]
...
```

Usually `vmbr0` is the `[PUBLIC_INTERFACE]`.

To make the changes live without restarting the interface or rebooting the node you can execute the iptables command manually.

#### To enable port forwarding

```
iptables -t nat -A PREROUTING -i [PUBLIC_INTERAFCE] -p [TCP|UDP] --dport [EXTERNAL_PORT] -j DNAT --to [INTERNAL_IP]:[INTERNAL_PORT]
```

#### To disable port forwarding

```
post-down iptables -t nat -D PREROUTING -i [PUBLIC_INTERAFCE] -p [TCP|UDP] --dport [EXTERNAL_PORT] -j DNAT --to [INTERNAL_IP]:[INTERNAL_PORT]
```

You can also use a range of ports (eg. `--dport 3300:3500`) if they are going to match the internal ports (`--to 192.168.0.100`).

## Examples

### NGINX Reverse Proxy

For nginx's container, I'm going with 35565 as the SSH port. I'm also going to forward port 80 and 443.

#### /etc/network/interfaces

```
...
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 35565 -j DNAT --to 192.168.0.100:22
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 35565 -j DNAT --to 192.168.0.100:22
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to 192.168.0.100:80
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to 192.168.0.100:80
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to 192.168.0.100:443
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to 192.168.0.100:443
...
```

```
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 35565 -j DNAT --to 192.168.0.100:22
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 80 -j DNAT --to 192.168.0.100:80
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 443 -j DNAT --to 192.168.0.100:443
```

### Plex Media Server

Plex Media Server not only requires TCP ports, but also some UDP ports. Fortunately, it's as simple as changing the `tcp` to `udp` in the command. This time 30053 is going to be the SSH port.

TCP: 3005; 32400; 8324; 32469 UDP: 1900; 5353; 32410

#### /etc/network/interfaces

```
...
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 30053 -j DNAT --to 192.168.0.101:22
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 30053 -j DNAT --to 192.168.0.101:22
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 3005 -j DNAT --to 192.168.0.101:3005
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 3005 -j DNAT --to 192.168.0.101:3005
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 32400 -j DNAT --to 192.168.0.101:32400
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 32400 -j DNAT --to 192.168.0.101:32400
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 8324 -j DNAT --to 192.168.0.101:8324
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 8324 -j DNAT --to 192.168.0.101:8324
post-up iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 32469 -j DNAT --to 192.168.0.101:32469
post-down iptables -t nat -D PREROUTING -i vmbr0 -p tcp --dport 32469 -j DNAT --to 192.168.0.101:32469
post-up iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 1900 -j DNAT --to 192.168.0.101:1900
post-down iptables -t nat -D PREROUTING -i vmbr0 -p udp --dport 1900 -j DNAT --to 192.168.0.101:1900
post-up iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 5353 -j DNAT --to 192.168.0.101:5353
post-down iptables -t nat -D PREROUTING -i vmbr0 -p udp --dport 5353 -j DNAT --to 192.168.0.101:5353
post-up iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 32410 -j DNAT --to 192.168.0.101:32410
post-down iptables -t nat -D PREROUTING -i vmbr0 -p udp --dport 32410 -j DNAT --to 192.168.0.101:32410
...
```

```
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 30053 -j DNAT --to 192.168.0.101:22
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 3005 -j DNAT --to 192.168.0.101:3005
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 32400 -j DNAT --to 192.168.0.101:32400
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 8324 -j DNAT --to 192.168.0.101:8324
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 32469 -j DNAT --to 192.168.0.101:32469
iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 1900 -j DNAT --to 192.168.0.101:1900
iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 5353 -j DNAT --to 192.168.0.101:5353
iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 32410 -j DNAT --to 192.168.0.101:32410
```

## That's it.

We are done. You can now follow [How to hide Proxmox's web interface behind reverse proxy](https://new.mrpsycho.pl/cheatsheets/Hide-Proxmox-interface-behind-nginx-reverse-proxy-SSL-VNC/).
