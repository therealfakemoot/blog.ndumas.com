---
title: "Selinux and Nginx"
date: 2018-04-13T16:28:20Z

tags: ["selinux","nginx","fedora"]
series: ''
draft: false
---

# SELinux
DigitalOcean's Fedora droplets include SELinux. I don't know a great deal about SELinux but it's presumably a good thing for helping prevent privilege escalations and so on. Unfortunately, it can be troublesome when trying to do simple static site stuff with nginx.

## nginx
With Fedora and nginx and selinux all in use simultaneously, you are allowed to tell nginx to serve files that are owned/grouped under a user other than nginx's. This is phenomenally useful when working with something like hugo. This is possible because SELinux monitors/intercepts syscalls relating to file access and approves/denies them based on context, role, and type. SELinux concepts are covered pretty thoroughly [here](https://www.digitalocean.com/community/tutorials/an-introduction-to-selinux-on-centos-7-part-1-basic-concepts) and [here](https://www.digitalocean.com/community/tutorials/an-introduction-to-selinux-on-centos-7-part-2-files-and-processes).

By default, nginx runs under the SELinux `system_u` user, the `system_r` role, and the `httpd_t` type:

```
$ ps -efZ|grep 'nginx'
system_u:system_r:httpd_t:s0    root     30543     1  0 Apr09 ?        00:00:00 nginx: master process /usr/sbin/nginx
system_u:system_r:httpd_t:s0    nginx    30544 30543  0 Apr09 ?        00:00:02 nginx: worker process
system_u:system_r:httpd_t:s0    nginx    30545 30543  0 Apr09 ?        00:00:00 nginx: worker process
$
```

Roughly speaking, SELinux compares nginx's user, role, and type against the same values on any value it's trying to access. If the values conflict, SELinux denies access. In the context of "I've got a pile of files I want nginx to serve", this denial manifests as a 403 error. This has caused issues for me repeatedly. genesis generates terrain renders as directories containing html and json files, and during the development and debugging process I just copy these directly into the `/var/www` directory for my renders.ndumas.com subdomain. Before I discovered the long term fix described below, every one of these new pages was throwing a 404 because this directory and its files did not have the `httpd_sys_content_t` type set. This caused nginx to be denied permission to read them and a 403 error.

A correctly set directory looks like this:

```
$ ls -Z
unconfined_u:object_r:httpd_sys_content_t:s0 demo/                 unconfined_u:object_r:user_home_t:s0 sampleTest2/  unconfined_u:object_r:httpd_sys_content_t:s0 test2/
unconfined_u:object_r:httpd_sys_content_t:s0 sampleTest1/  unconfined_u:object_r:httpd_sys_content_t:s0 test1/
$
```

# The solution
There are two solutions to serving static files in this way. You can set the `httpd_sys_content_t` type for a given directory and its contents, or you can alter SELinux's configuration regarding the access of user files.

## Short Term
The short term fix is rather simple: `chcon -R -t httpd_sys_content_t /var/www/`. This sets a type value on the directory and its contents that tells SELinux that the `httpd_t` process context can read those files.

## Long Term
Unfortunately, in the context of my use case, I had to run that `chcon` invocation every time I generated a new page. I hate manual labor, so I had to find a way to make this stick. Fortunately, [StackOverflow](https://stackoverflow.com/questions/22586166/why-does-nginx-return-a-403-even-though-all-permissions-are-set-properly#answer-26228135) had the answer.

You can tell SELinux "httpd processes are allowed to access files owned by other users" with the following command: `setsebool -P httpd_read_user_content 1`. This is pretty straightforward and I confirmed that any content I move into the /var/www directories can now be read by nginx.
