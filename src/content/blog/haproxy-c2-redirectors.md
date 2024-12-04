---
title: Configuring HAProxy for C2 Redirection
author: taylor
pubDatetime: 2024-12-03T13:53:49Z
slug: haproxy-c2-redirectors
featured: false
draft: true
tags:
  - c2
  - infrastructure
  - red-vs-blue
description: Using HAProxy's ACL feature to redirect beacon traffic to C2 servers.
---

## Table of Contents

## Why redirectors?

(Because we practice safe red teaming)

C2 redirectors serve three main purposes:

- Reduces exposure of our C2 servers from the public, which can contain sensitive info
- Masks the "true" source of our attacking infrastructure
- Averts traffic to our attacking infrastructure from prying eyes, such as from web scrapers and threat hunters

## What is HAProxy?

[High Availability Proxy (HAProxy)](https://www.haproxy.org/) is an open-source reverse-proxy tool used on Linux systems. It's meant to improve performance and availability of web servers.

But it offers another option besides `.htaccess` files for routing web traffic to C2 servers.

HAProxy contains two main components:

- A "frontend" that acts as the entry point and handles incoming connections
- A "backend" that defines the servers that HAProxy will forward the requests

HAProxy can have multiple frontends and backends, if you want to listen on multiple ports and redirect traffic to multiple hosts.

## What does your C2 traffic look like?

In order to redirect traffic, we need to know what kind of traffic we want to redirect in the first place. For web-based C2 listeners, that usually comes in the form of URIs, request headers, type of request (GET, POST), and more.

My infra:

- Redirector box: `10.2.99.10`
- Mythic C2 server: `10.2.99.99`

I configured my payload to use these settings (note that the callback host is set to the IP of the redirector):

![](https://i.postimg.cc/hjxZ6RBr/SCR-20241203-pknz.png)

## HAProxy configuration file setup

Before we set up HAProxy, we must first install HAProxy.

```bash
sudo apt install haproxy
```

Configure SSL certificate stuff as well:

```bash
sudo apt install ssl-cert
cp /etc/ssl/certs/ssl-cert-snakeoil.pem /etc/haproxy
cp /etc/ssl/private/ssl-cert-snakeoil.key /etc/haproxy/ssl-cert-snakeoil.pem.key # haproxy is weird
```

To configure the frontend, we will set the mode to `http` and bind to ports 80 and 443. Our main C2 traffic will be using HTTPS, but we want our redirector to automatically redirect all HTTP traffic on port 80 to HTTPS as well.

```apache
frontend redirector
  mode http
  bind *:80
  bind *:443 ssl crt /etc/haproxy/ssl-cert-snakeoil.pem
  http-request redirect scheme https unless { ssl_fc }
```

Next up we define our ACLs. Our beacon callbacks are configured to use a specific user agent, as well as hit the `/pages` and `/info` endpoints on the C2 server. We can create ACLs for those:

```apache
  # format: acl <name> <parameter to match> <flags> <values>
  # -i = case insensitive
  # -m = match type

  # match User-Agent request header to a specific value
  acl ua req.fhdr(User-Agent) -i "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.5481.100 Safari/537.36"

  # match URL paths that begin with /pages or /info
  acl urls path -i -m beg /pages /info
```

Now we define the action to take if ACLs match.

```apache
  # format: use_backend <backend name> if <condition>
  use_backend c2 if ua urls
```

We set a default backend as a catch-all for non-matching traffic:

```apache
  # format: default_backend <backend-name>
  default_backend catchall
```

Now let's set our backends. We have 2 backends: one for the C2 server and the other for all other traffic.

```apache
# backend for C2 traffic
backend c2
  mode http
  server mythic 10.2.99.99:443 ssl verify none

# backend going to a locally hosted Apache2 server
backend catchall
  mode http
  server bruh 127.0.0.1:8080
```

For troubleshooting purposes, let's enable logging.

```apache
# add on the very top
global
  log /dev/log local0
  log /dev/log local1 notice

# add in frontend section
  log global
  option httplog

```

### HAProxy full config file

Here's the full config file:

```apache
global
  log /dev/log local0
  log /dev/log local1 notice

frontend redirector
  mode http
  bind *:80
  bind *:443 ssl crt /etc/haproxy/ssl-cert-snakeoil.pem
  http-request redirect scheme https unless { ssl_fc }

  # Enable logging
  log global
  option httplog

  # Define ACLs
  acl ua req.fhdr(User-Agent) -i "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.5481.100 Safari/537.36"
  acl urls path -i -m beg /pages /info

  # Forwrd to C2 if conditions are met
  use_backend c2 if ua urls

  # If no ACL matches, redirect to a different landing page
  default_backend catchall

backend c2
  mode http
  server mythic 10.2.99.99:443 ssl verify none

backend catchall
  mode http
  server bruh 127.0.0.1:8080

```

## Setting up the catch-all backend web server

Set up Apache2 to listen on non-default port (e.g. 8080) and host a static webpage on there.

Install Apache2 if not installed already:

```bash
sudo apt install apache2
```

Change `Listen 80` to `Listen 8080` in `/etc/apache2/ports.conf` file:

```apache
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

Listen 8080

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>
```

Change `<VirtualHost *:80>` to `<VirtualHost *:8080>` in `/etc/apache2/sites-available/000-default.conf`:

```apache
<VirtualHost *:8080>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com
```

Restart Apache2:

```bash
sudo systemctl restart apache2
```

Apache2 should now be listening on port 8080.

## Running HAProxy

Typically I like to run it in a tmux session.

```bash
haproxy -f redirector.conf
```

## Troubleshooting

If you aren't seeing any traffic come back, check your HAProxy logs. Make sure that `rsyslog` is installed first:

```bash
sudo apt install rsyslog
```

You can then `tail /var/log/haproxy.log`.

The below screenshot is a snippet of the generated log file. The first line is C2 traffic incorrectly being redirected to a default Apache2 landing page, and the second line is after fixing a few things, where the C2 traffic is being correctly routed to the C2 server.

![](https://i.postimg.cc/d1mmGsFz/SCR-20241203-posg.png)
