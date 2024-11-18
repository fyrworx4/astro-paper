---
title: Configuring HAProxy for C2 Redirection
author: taylor
pubDatetime: 2024-11-18T05:22:49Z
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

Because we practice safe red teaming

C2 redirectors serve two main purposes:

- Masks the real IP address of C2 team servers
- Averts traffic that does not match the profile used by the C2 team server, such as traffic from web scrapers and threat hunters

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

- Redirector box: 172.16.1.1
- Mythic C2 server: 172.16.9.143

For example, my HTTP C2 Profile in Mythic looks like this:

insert stuff here

I configured my payload to use these settings:

insert sc here

## HAProxy configuration file setup

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
  acl ua req.hdr(User-Agent) -i "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.5481.100 Safari/537.36"
  acl urls path -i -m beg /pages /info

  # Forwrd to C2 if conditions are met
  use_backend c2 if urls

  # If no ACL matches, use pfSense landing pages
  default_backend pfsense

backend c2
  mode http
  server mythic 172.16.9.143:443 ssl verify none

backend pfsense
        mode http
        server bruh 127.0.0.1:8080

```

## Running HAProxy

Typically I like to run it in a tmux session.

```bash
haproxy -f redirector.conf
```
