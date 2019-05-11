---
title: Bash script for securing SSH
date: 2019-05-11 14:47:12
tags:
  - SSH
  - Bash
categories:
  - cheatsheets
author: Jakub Papuga
---
## What does this script do?

It chooses a random port between 49151 and 65535 (so-called dynamic ports, so the SSH won't collide with any application) and makes SSH listen on that port. This script also completely disables password authentification, so the only option is to use SSH keys. No one is going to brute force that. It can also create a master key, which grants you access to all users. It is useful in private environments, but be careful with it.

```bash
wget https://i.mrpsycho.pl/selif/securessh.sh && bash securessh.sh
```

```bash
#!/bin/bash
CONF_FILE=/etc/ssh/sshd_config
x=$(($RANDOM+49151))
while [ $x -gt 65535 ]
do
  x=$(( $RANDOM+49151 ))
done

sed -i "/Port/c\Port $x" $CONF_FILE
sed -i '/ChallengeResponseAuthentication/c\ChallengeResponseAuthentication no' $CONF_FILE
sed -i '/PasswordAuthentication/c\PasswordAuthentication no' $CONF_FILE
sed -i '/UsePAM/c\UsePAM no' $CONF_FILE
sed -i '/PermitRootLogin/c\PermitRootLogin without-password' $CONF_FILE

read -p "Enter your public SSH key: " pub_key

hashes="########################################"
ssh_num="Your randomised SSH port number is: $x"
use_all_yes="Keyless login disbaled. You can use the provided key to login as anyone."
use_all_no="Keyless login disabled. The provided key will only be used by the current user.\nYou can add more SSH keys for appropriate user in the ~/.ssh/authorized_keys file.\n"

while true; do
    read -p "Do you want to use one SSH key for all users? [y/n]: " yn
    case $yn in
        [Yy]* ) sed -i '/AuthorizedKeysFile/c\AuthorizedKeysFile %h/.ssh/authorized_keys /etc/authorized_keys' /etc/ssh/sshd_config && echo $pub_key >> /etc/authorized_keys && chmod 444 /etc/authorized_keys && echo $hashes && echo $ssh_num && echo $hashes && echo $use_all_yes; break;;
        [Nn]* ) if [ ! -d "$HOME/.ssh" ]
                then
                  mkdir $HOME/.ssh
                fi
                echo $pub_key >> ~/.ssh/authorized_keys && echo $ssh_num && printf "$use_all_no"; break;;
        * ) echo "Please answer yes or no. [y/n]: ";;
    esac
done

while true; do
    read -p "Do you want to reload the SSH config? [y/n]: " yn
    case $yn in
        [Yy]* ) service ssh reload; exit;;
        [Nn]* ) exit;;
        * ) echo "Please answer yes or no.";;
    esac
done
```

## Some explanation
The script searches for the lines containing specific strings like `Port` or `PasswordAuthentication` in the `/etc/ssh/sshd_config` and replaces them in whole, so there is no way of creating a bad config (eg. leaving `#` in front of the line or appending the text)
The master key is created with read only permission and is located under `/etc/master_key`. The script also changes both `ChallengeResponseAuthentication` and `UsePAM` to no, as setting just `PasswordAuthentication` is not enough as the previously mentioned settings would still allow for password login.
Here is a good a good explaination from [Izzy](https://superuser.com/questions/161609/can-someone-explain-the-passwordauthentication-in-the-etc-ssh-sshd-config-fil/374234#374234).
