---
title: 'Tinc Centralized Interconnection'
tags:
  - VPN
  - Encryption
categories:
  - cheatsheets
author: Jakub Papuga
date: 2019-03-13 12:54:00
---
## Introduction

This time we are going to use Node C as our traffic aggregator.
You should already be familar with basic tinc configuration, as I described in this previous article - [Configuring Tinc, an encrypted P2P VPN](https://new.mrpsycho.pl/cheatsheets/how-to-configure-tinc-peer-to-peer-vpn).

## Topology cheatsheet

|  | Node A | Node B | Node C |
|:---|:---:|:---:|:---:|
| Public IPv4 | 1.1.1.1 | 2.2.2.2 | 3.3.3.3 |
| VPN Address | 10.3.0.1 | 10.3.0.2 | 10.3.0.3 |
| VPN Network Name | PsychoVPN | PsychoVPN | PsychoVPN |


## Node C configuration files

Create the working directory to store the configuration:

```
mkdir -p /etc/tinc/PsychoVPN/hosts && cd /etc/tinc/PsychoVPN
```

Start off with editing `tinc.conf`:

#### Node C: /etc/tinc/PsychoVPN/tinc.conf

```
Name = NodeC
Device = /dev/net/tun
AddressFamily = ipv4
ConnectTo = NodeA
ConnectTo = NodeB
```

### Restricting Node A and B

Modify the `tinc.conf` on both Node A and B, so that they point only to Node C:

#### Node A: /etc/tinc/PsychoVPN/tinc.conf

```
Name = NodeA
Device = /dev/net/tun
AddressFamily = ipv4
ConnectTo = NodeC
```

#### Node B: /etc/tinc/PsychoVPN/tinc.conf

```
Name = NodeB
Device = /dev/net/tun
AddressFamily = ipv4
ConnectTo = NodeC
```

Create a host file for Node C:

#### Node C: /etc/tinc/PsychoVPN/hosts/NodeC

```
Address = 3.3.3.3
Subnet = 10.3.0.3
```

Create a pair of keys (make sure to save both files under `/etc/tinc/PsychoVPN`):

```
tincd -n PsychoVPN -K 4096
```

Append the public key to `NodeC`'s host file.

```
cat rsa_key.pub >> hosts/NodeC
```

Now you can exchange the host file with Node A and Node B

```
scp /etc/tinc/PsychoVPN/hosts/NodeC <user>@<NodeA>:/etc/tinc/PsychoVPN/hosts/NodeC
scp /etc/tinc/PsychoVPN/hosts/NodeC <user>@<NodeB>:/etc/tinc/PsychoVPN/hosts/NodeC
```

It's also required for Node C to have both Node A and Node B host files. To you reverse the `scp` to download files from Node A and Node B directly from Node C:

```
scp <user>@<NodeA>:/etc/tinc/PsychoVPN/hosts/NodeA /etc/tinc/PsychoVPN/hosts/NodeA
scp <user>@<NodeB>:/etc/tinc/PsychoVPN/hosts/NodeB /etc/tinc/PsychoVPN/hosts/NodeB
```

You can also just copy the content of `hosts` folder from a different node, saving you the hassle of double authentication:

```
scp <user>@<NodeA>:/etc/tinc/PsychoVPN/hosts/* /etc/tinc/PsychoVPN/hosts/
```

### That's it!

From this point just follow the previous instructions starting from [#Control Scripts](https://new.mrpsycho.pl/cheatsheets/how-to-configure-tinc-peer-to-peer-vpn/#Control-Scripts). Just remember to adjust the IP addresses correspondingly.
