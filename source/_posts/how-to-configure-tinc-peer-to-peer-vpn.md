title: 'Configuring Tinc, an encrypted P2P VPN'
tags:
  - VPN
  - Encryption
categories:
  - cheatsheets
author: Jakub Papuga
date: 2019-02-15 13:34:00
---
## What is tinc?
Tinc is a dead simple , yet super flexible VPN deamon.

It also has some nice features, such as:
+ encryption
+ automatic mesh routing
+ compression
+ NAT traversal
+ IPv6 support

### This articale will demonstrate how to create a basic two node VPN and how to add new nodes to the mesh network.

## Topology cheatsheet

|  | Node A | Node B |
|:---|:---:|:---:|
| Public IPv4 | 1.1.1.1 | 2.2.2.2 |
| VPN Address | 10.3.0.1 | 10.3.0.2 |
| VPN Network Name | PsychoVPN | PsychoVPN |

## Updating the OS

As always, start with refreshing your package list on both nodes.

```
apt update
```

...and preferably upgrade the packages

```
apt upgrade -y
```

### Now you can go ahead and install tinc:

```
apt install tinc -y
```

## Creating Configuration Files

### Main Configuration File

Tinc requires a main configuration file named `tinc.conf`. The folder in which you put the `tinc.conf` file has to match the designated name of your VPN - in my case: `/etc/tinc/PsychoVPN`.

```
mkdir -p /etc/tinc/PsychoVPN/hosts
```

Tinc has a lot of configurable options

+ `Name` - this is our node name within the network   
+ `Device` - Determinaes the virtual network to use.
+ `AddressFamily` - indicates which type of address to use (ipv4|ipv6|any)
+ `ConnectTo` - Specifies which other tinc daemon to connect to on startup.  Multiple ConnectTo variables may be specified, in which case outgoing connections to each specified tinc daemon are made.  The names should be known to this tinc daemon (i.e., there should be a host configuration file for the name on the ConnectTo line). Gives the ability to create a centralized interconnection (make NodeA and NodeB connect only to NodeC and from there make NodeC connect to both servers). If you don't specify a host with ConnectTo, tinc won't try to connect to other daemons at all, and will instead just listen for incoming connections.
+ `BindToAddress = [address] [port]` - If your computer has more than one IPv4 or IPv6 address, tinc will by default listen on all of them for incoming connections.  Multiple BindToAddress variables may be specified. If no port is specified, the socket will be bound to the port specified by the Port option, or to port 655 if neither is given.  To only bind to a specific port but not to a specific address, use * for the address.
+ `BindToInterface = [interface]` - If your computer has more than one network interface, tinc will by default listen on all of them for incoming connections.  It is possible to bind only to a single interface with this variable.
+ More optins avaiable [here](https://www.tinc-vpn.org/documentation/tinc.conf.5)

Here is a simple main configuration for NodeA:

#### Node A: /etc/tinc/PsychoVPN/tinc.conf

```
Name = NodeA
Device = /dev/net/tun
AddressFamily = ipv4
ConnectTo = NodeB
```

...and here is for NodeB

#### Node B: /etc/tinc/PsychoVPN/tinc.conf

```
Name = NodeB
Device = /dev/net/tun
AddressFamily = ipv4
ConnectTo = NodeA
```

### Host configuration files

Because tinc is a P2P VPN it needs to communicate with the other servers. The very basic host file looks like this and has to be named afer the node name:

```
Address = 
Subnet = 
```

We also have to add public key of specific node to that file.

So on the Node A we are going to create a `NodeA` file under the `../hosts` folder.

#### Node A: /etc/tinc/PsychoVPN/hosts/NodeA

```
Address = 1.1.1.1
Subnet = 10.3.0.1
```

...and similarly on the Node B (`../hosts/NodeB`) 

#### Node B: /etc/tinc/PsychoVPN/hosts/NodeB

```
Address = 2.2.2.2
Subnet = 10.3.0.2
```

Now we have to create a 4096-bit pair of keys on each node

```
tincd -n PsychoVPN -K 4096
```

Make sure to save both `rsa_key.pub` and `rsa_key.priv` to the root of your working directory (`/etc/tinc/PsychoVPN`)

Now we can append those public keys to the corresponding hosts files

#### Node A:

```
cat rsa_key.pub >> hosts/NodeA
```

#### Node B:

```
cat rsa_key.pub >> hosts/NodeB
```

Now we should exchange the hosts files between the nodes. You can use scp for example:

From NodeA

```
scp /etc/tinc/PsychoVPN/hosts/NodeA <user>@<NodeB>:/etc/tinc/PsychoVPN/hosts/NodaA
```

From NodeB

```
scp /etc/tinc/PsychoVPN/hosts/NodeB <user>@<NodeA>:/etc/tinc/PsychoVPN/hosts/NodaB
```

## Control Scripts

Control scripts are responsible for setting up virtual interfaces on each server. They’re needed on both servers.

Script for enabling tinc interface on Node A:

#### Noda A: /etc/tinc/PsychoVPN/tinc-up

```
#!/bin/sh
ip link set $INTERFACE up
ip addr add 10.3.0.1 dev $INTERFACE
ip route add 10.3.0.0/24 dev $INTERFACE
```

Script for disabling tinc interface on Node A:

#### Node A: /etc/tinc/PsychoVPN/tinc-down

```
#!/bin/sh
ip route del 10.3.0.0/24 dev $INTERFACE
ip addr del 10.3.0.1 dev $INTERFACE
ip link set $INTERFACE down
```

...enabling tinc interface on Node B:

#### Node B: /etc/tinc/PsychoVPN/tinc-up

```
#!/bin/sh
ip link set $INTERFACE up
ip addr add 10.3.0.2 dev $INTERFACE
ip route add 10.3.0.0/24 dev $INTERFACE
```

...disabling tinc interface on Node B:

#### Node B: /etc/tinc/PsychoVPN/tinc-down

```
#!/bin/sh
ip route del 10.3.0.0/24 dev $INTERFACE
ip addr del 10.3.0.2 dev $INTERFACE
ip link set $INTERFACE down
```

After creating the control scrips, you have to adjust the permissions on both servers:

```
chmod -v +x /etc/tinc/PsychoVPN/tinc-{up,down}
```

## Testing tinc

Your newly configured VPN should be ready to test. Start the deamon on both servers:

```
tincd -n PsychoVPN -LD -d3
```

Now you should be able to ping Node B from Node A and vice versa from new terminal. To elevate debug level from 3 to 5 press `CTRL+C`. To stop tincd press `CTRL+\`. If you ran tincd in background you can use the `-k` option to kill the running tincd.

## Run tinc on startup

To run tinc on startup you will need to set up a systemd unit file on each node.

#### /etc/systemd/system/PsychoVPN.service
```
[Unit]
Description=Tinc PsychoVPN Encrypted P2P Network
After=network.target

[Service]
Type=simple
WorkingDirectory=/etc/tinc/PsychoVPN
ExecStart=/usr/sbin/tincd -n PsychoVPN -LD -d2
ExecStop=/usr/sbin/tincd -n PsychoVPN -k
TimeoutStopSec=5
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

Now enable the service on startup

```
sudo systemctl enable PsychoVPN.service
```

You can start, stop the service using

```
systemctl start|stop PsychoVPN.service
```
