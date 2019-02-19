---
title: 'NGINX Reverse Proxy with caching and SSL'
date: 2019-02-19 18:45:52
tags:
  - nginx
  - reverse proxy
  - caching
  - SSL
categories:
  - cheatsheets
author: Jakub Papuga
---
## Generic config 

#### /etc/nginx/sites-available/domain.tld

```
proxy_cache_path /path/to/cache levels=1:2 keys_zone=something_cache:10m max_size=2g inactive=15m use_temp_path=off;
server  {

  listen  80;
  server_name  domain.tld;
  return 301 https://$server_name/$1;
}

server  {

  listen 443;
  server_name  domain.tld;

  ssl  on;
  ssl_certificate  /path/to/fullchain.pem;
  ssl_certificate_key /path/to/privkey.pem;
  ssl_session_timeout  5m;

  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
  server_tokens off;

  access_log /var/log/nginx/domain.tld_access.log;
  error_log /var/log/nginx/domain.tld_error.log;


  location  / {

	proxy_pass http://content.remote;
	proxy_set_header X-Forwarded-for $proxy_add_x_forwarded_for;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header Host $host;
	proxy_hide_header X-Powered-By;
	add_header X-Cache-Status $upstream_cache_status;
	 
	proxy_cache something_cache;
	proxy_cache_revalidate on;
	proxy_cache_valid 10m;
	proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
	proxy_cache_background_update on;
	proxy_cache_min_uses 1;
	proxy_cache_lock on;
  }

}
```

To enable the new site run

```
ln -s /etc/nginx/sites-available/domain.tld /etc/nginx/sites-enabled/domain.tld \
&& nginx -t && service nginx reload
```

## Explanation

The first `server{}` block listening on port 80 simply redirects the traffic to HTTPS url.

`levels` - if not included NGINX puts all files in the same directory, slowing things down.

`keys_zone` - defines a place in memory where NGINX stores metadata greately imporivng performance. 1MB zone can store roughly 8000 entries.

`inactive` - defines how long an item can remain in the cache without being accessed. Expired (stale) content is deleted only when it has not been accessed for the time specified by that paramer.

`Strict-Transport-Security` - once the first page is loaded over HTTPS, the brower will prevent you from communicating with the server over HTTP for 31536000 seconds (1 year)

`add_header X-Cache-Status` - reports the cahce status (MISS|BYPASS|EXPIRED|STALE|UPDATING|REVALIDED|HIT) in X-Cahce-Status header.

`proxy_cache_bypass   $http_secret_header;` - if any client fires a HTTP request that contains header `Secret-Header: 1`, nginx will bypass the request regardless of whether the requested resource was cached or not.

`proxy_cache_valid` - for how long the cache should be valid

`proxy_cache_use_stale` - if NGINX receives an error, timeout, any of the specified 5xx errors or is updating the cache in the background from the origin server and it has a stale version of the requested file in its cache, it delivers the stale file.

`proxy_cache_min_uses` - sets the number of times an item must be requested by clients before NGINX caches it. This is useful if the cache is constantly filling up, as it ensures that only the most frequently accessed items are added to the cache.

`proxy_cache_lock` - if multiple clients request a file that is not current in the cache (a MISS), only the first of those requests is allowed through to the origin server. The remaining requests wait for that request to be satisfied and then pull the file from the cache. Without proxy_cache_lock enabled, all requests that result in cache misses go straight to the origin server.

## Config that is used for this blog

```
proxy_cache_path /var/www/personal-website/cache levels=1:2 keys_zone=personal_website:10m max_size=500m inactive=60y use_temp_path=off;
server  {

  listen  80;
  server_name  new.mrpsycho.pl;
  return 301 https://$server_name/$1;
}

server  {

  listen 443;
  server_name  new.mrpsycho.pl;

  ssl  on;
  ssl_certificate  /etc/letsencrypt/live/mrpsycho.pl-0002/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/mrpsycho.pl-0002/privkey.pem;
  ssl_session_timeout  5m;

  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
  server_tokens off;

  access_log /var/log/nginx/new.mrpsycho.pl_access.log;
  error_log /var/log/nginx/new.mrpsycho.pl_error.log;


  location  / {

	proxy_pass http://192.168.0.126:4000;
	proxy_set_header X-Forwarded-for $proxy_add_x_forwarded_for;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header Host $host;
	proxy_hide_header X-Powered-By;
	add_header X-Cache-Status $upstream_cache_status;

	proxy_cache personal_website;
	proxy_cache_revalidate on;
	proxy_cache_valid 30d;
	proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
	proxy_cache_background_update on;
	proxy_cache_min_uses 1;
	proxy_cache_lock on;
	proxy_cache_bypass $http_secret_header;
  }

}
```
