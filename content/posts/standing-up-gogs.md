---
title: "Standing Up Gogs"
date: 2018-02-20T23:57:30Z
tags: ["nginx"]
draft: false
---

# The Reveal

It took me way longer than I'd like to admit but I finally discovered why I was not able to properly reverse proxy a subdomain ( [git.ndumas.com](http://git.ndumas.com) ). As it turns out, SELinux was the culprit. I wanted to write a short post about this partly to reinforce my own memory and maybe to make this sort of thing easier to find for others encountering 'mysterious' issues when operating on SELinux Fedora installations like Digital Ocean droplets.

# Symptoms

SELinux interference with nginx doing its business will generally manifest as a variation on "permission denied". Here's one such error message:

```
2018/02/20 23:32:51 [crit] 4679#0: *1 connect() to 127.0.0.1:3000 failed (13: Permission denied) while connecting to upstream, client: xxx.xxx.xxx.xxx, server: git.ndumas.com, request: "GET /favicon.ico HTTP/1.1", upstream: "http://127.0.0.1:3000/favicon.ico", host: "git.ndumas.com", referrer: "http://git.ndumas.com/"
```

# Solution

Resolving this requires whitelisting whatever action/context SELinux is restricting. There's a useful tool, `audit2allow`, that will build a module for you automatically. You can invoke it like this:

```
sudo cat /var/log/audit.log|grep nginx|grep denied|audit2allow -M filename
```

Once you've done this, you'll have a file named `filename.pp`. All you have to do next is:

```
semodule -i filename.pp
```

SELinux will revise its policies and any restrictions nginx encountered will be whitelisted.
