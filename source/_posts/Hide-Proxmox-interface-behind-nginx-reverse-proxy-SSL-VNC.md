---
title: Hiding Proxmox 5 web interface behind reverse proxy with SSL
date: 2019-04-01 16:35:01
tags:
  - nginx
  - SSL
  - Proxmox
  - NAT
categories:
  - cheatsheets
author: Jakub Papuga
---
## Introduction 

In this article, I assume that you already know what reverse proxy is, how to use DNS. I'm also assuming you already have an nginx server. In the [last](https://new.mrpsycho.pl/cheatsheets/Proxmox-on-OVH-Kimsufi-behind-single-IP-NAT/) article I've explained how to configure Proxmox to work with one IPv4 and as an example, I used a container with nginx, so you may want to take a look at it if you want to put the reverse proxy on a node with one IPv4. Of course, you can use whatever server with nginx you want (preferably one with low latency to the host node). I only advise you against installing nginx directly on the host node as you will lose the ability to effortlessly backup and restore the service as if it was in a VM. Furthermore installing additional software on the host node is generally a bad practice.

In a separate article, I'll describe how to restrict Proxmox interface geographically. If the article is available I suggest you reading it first, as geographic blocking requires compiling nginx with Geo-IP module. 

## Let's Encrypt 

Firstly we have to obtain an SSL certificate. Presumably, also automate its renewal. Since there are a few different ways to install cerbot, I'm going with the one that is the most universal. Execute the following command inside the container.

Download certbot-auto script and make it executable:

```
wget https://dl.eff.org/certbot-auto && chmod a+x certbot-auto
```

Now run it. Certbot will automagically download and install all its dependencies.

```
./certbot-auto
```

Proceed with the configurator.

```
Enter email address (used for urgent renewal and security notices) (Enter 'c' to
cancel): email@domain.tld

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel: a

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: n
No names were found in your configuration files. Please enter in your domain
name(s) (comma and/or space separated)  (Enter 'c' to cancel): proxmox.domain.tld
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for proxmox.domain.tld
Waiting for verification...
Cleaning up challenges
Deploying Certificate to VirtualHost /etc/nginx/sites-enabled/default

Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 1
```

We're going to use our own nginx configuration, so there is no need for certbot to make any adjustments.

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/proxmox.domain.tld/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/proxmox.domain.tld/privkey.pem
   Your cert will expire on 2019-06-25. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again with the "certonly" option. To non-interactively renew *all*
   of your certificates, run "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

### Automation

Certbot now has an nginx plugin which gives the ability to obtain the certificate without interrupting nginx, so there is no longer a need to stop nginx and use the standalone server. To renew the certificates on every 3rd day of the month at 2:00AM simply run `crontab -e` and add the following:

```
0 2 3 * * /path/to/certbot-auto renew
```

## NGINX Reverse Proxy

To make VNC work on Proxmox 5.x we have to do some magic and copy some folders from the host node as nginx hangs while waiting for them. To copy the folders painlessly mount the container filesystem on the host. This will put a lock on the container.

```
pct mount 100
```

Now copy the following directories. The first half (till /var/www) of the second path is the path given you by `pct mount`

```
mkdir /var/lib/lxc/100/rootfs/var/www/proxmox/
cp -r /usr/share/pve-manager /var/lib/lxc/100/rootfs/var/www/proxmox/pve-manager/
mkdir /var/lib/lxc/100/rootfs/var/www/proxmox/javascript/
cp -r /usr/share/javascript/proxmox-widget-toolkit /var/lib/lxc/100/rootfs/var/www/proxmox/javascript/proxmox-widget-toolkit
cp -r /usr/share/pve-docs /var/lib/lxc/100/rootfs/var/www/proxmox/pve-docs
```

Now set the permissions, so that the container can do anything with those files and umount the filesystem. For the user & group you can `ls -lh` any folder inside the file system (eg. `ls -lh /var/lib/lxc/100/rootfs/var/www/`)

```
chown -R 100000:100000 /var/lib/lxc/100/rootfs/var/www/proxmox
pct unmount 100
```

Now we switch back to the container and set the correct ownership of /var/www/proxmox

```
chown -R www-data:www-data /var/www/proxmox
```


Now create a new file under `/etc/nginx/sites-available/`

```
touch /etc/nginx/sites-available/proxmox.domain.tld
```

...and edit it. Make sure to change `proxmox.domain.tld` to your domain and the IP address to your Proxmox server.

#### /etc/nginx/sites-available/proxmox.domain.tld

```
server  {
  ## Redirect HTTP traffic to HTTPS
  listen  80;
  server_name proxmox.domain.tld;
  return 301 https://$server_name/$1;
}

server {
  listen 443 ssl http2;
  server_name proxmox.domain.tld;
  
  ssl_certificate  /etc/letsencrypt/live/proxmox.domain.tld/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/proxmox.domain.tld/privkey.pem;
  ssl_session_timeout  5m;

  valid_referers none blocked server_names;
  if ($invalid_referer) {
   return 403;
  }
  
  add_header X-Frame-Options SAMEORIGIN;
  add_header X-Content-Type-Options nosniff;
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
  server_tokens off;
        
  access_log /var/log/nginx/proxmox.domain.tld-access.log;
  error_log /var/log/nginx/proxmox.domain.tld-error.log;
  
  # Backups, images and ISO uploading
  client_max_body_size 5G;

  include proxy_params;
  

  location / {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    include proxy_params;
    proxy_pass https://192.168.0.254:8006;
  }

  location ~* ^/(api2|novnc)/ {
    proxy_redirect off;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    include proxy_params;
    proxy_pass https://192.168.0.254:8006;
  }
  
  #Some magic
  
  location ~* ^/pve2/(?<file>.*)$ {
    gzip_static on;
    root /var/www/proxmox/pve-manager/;
                try_files /$file @proxmox;
  }
  location ~* ^/proxmox.*\.js$ {
    gzip_static on;
    root /var/www/proxmox/javascript/proxmox-widget-toolkit;
    try_files $uri @proxmox;
  }
  location ~* ^/pve-docs/(?<file>.*)$ {
    gzip_static on;
    root /var/www/proxmox/pve-docs;
    try_files /$file @proxmox;
  }
  location @proxmox {
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    include proxy_params;
    proxy_pass https://192.168.0.254:8006;
        }
}
```

Now enable the site by linking the file inside `sites-available/` to `sites-enabled/`

```
ln -s /etc/nginx/sites-available/proxmox.domain.tld /etc/nginx/sites-enabled/proxmox.domain.tld
```

You can check for any errors by issuing `nginx -t` (if you get 'conflicting server name' that's because you still have the default_server inside sites-enabled/default)

If everything is okay reload the server

```
service nginx reload
```

## Finishing touches

Create the `/etc/default/pveproxy` file and edit it accordingly. This will make pveproxy only respond to our nginx reverse proxy making it unreachable from the internet.

#### /etc/default/pveproxy

```
ALLOW_FROM="192.168.0.100"
DENY_FROM="all"
POLICY="allow"
```

Now reload pveproxy:

```
service pveproxy reload
```

That's it. You're done.
