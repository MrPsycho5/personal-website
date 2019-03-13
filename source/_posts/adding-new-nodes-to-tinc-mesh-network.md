---
title: 'Adding new nodes to Tinc mesh network'
tags:
  - VPN
  - Encryption
categories:
  - cheatsheets
author: Jakub Papuga
date: 2019-03-13 12:34:00
---
## Introduction

This time we are going to add third node - Node C to the mesh network. Tinc is going automatically choose the best route.
You should already be familar with basic tinc configuration, that I described in previous article - [Configuring Tinc, an encrypted P2P VPN](https://new.mrpsycho.pl/cheatsheets/how-to-configure-tinc-peer-to-peer-vpn).

## Topology cheatsheet

|  | Node A | Node B | Node C |
|:---|:---:|:---:|:---:|
| Public IPv4 | 1.1.1.1 | 2.2.2.2 | 3.3.3.3 |
| VPN Address | 10.3.0.1 | 10.3.0.2 | 10.3.0.3 |
| VPN Network Name | PsychoVPN | PsychoVPN | PsychoVPN |

### Node C configuration files

Create the working directory:

```
mkdir -p /etc/tinc/PsychoVPN/hosts && cd /etc/tinc/PsychoVPN
```

Start off with `tinc.conf`:

#### Node C: /etc/tinc/PsychoVPN/tinc.conf

```
Name = NodeC
Device = /dev/net/tun
AddressFamily = ipv4
ConnectTo = NodeA
ConnectTo = NodeB
```

Now the host file:

#### Node C: /etc/tinc/PsychoVPN/hosts/NodeC

```
Address = 3.3.3.3
Subnet = 10.3.0.3
```

Create a pair of keys (make sure to save both files under `/etc/tinc/PsychoVPN`): 

```
tincd -n PsychoVPN -K 4096
```

Append the public key to `NodeC` host file.

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

You can also just copy the contexct of `hosts` folder from desired Node, saving you the hassle of double authentication:

```
scp <user>@<NodeA>:/etc/tinc/PsychoVPN/hosts/* /etc/tinc/PsychoVPN/hosts/
```

### That's it!

From this point just follow the previous aritcle's instructions starting from [#Control Scrips](https://new.mrpsycho.pl/cheatsheets/how-to-configure-tinc-peer-to-peer-vpn/#Control-Scripts). Just remember to adjust the IP addresses correspondingly!
