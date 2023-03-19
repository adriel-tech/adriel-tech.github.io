---
layout: post
title: "FreeBSD 13 BastilleBSD Tips"
categories: [BastilleBSD, FreeBSD13]
---

These are a few tricks I use for [BastilleBSD](https://bastillebsd.org/) on FreeBSD 13. This document assumes you have a home server on your lan, you want to use IPv6, your IPv4 and IPv6 are both dynamic addresses. You have installed BastilleBSD and are using ZFS. This setup can easily be used for public servers or laptops, use my examples to improve your own setup.

[1. Preamble information](#preamble-information)

[2. Host settings](#host-settings)

[3. Templates](#templates)

# Preamble information

- I mount my '.dotfiles' for jails in a shared folder. This allows a simple way to update userland
configs across all jails. I use [Xstow](https://www.freshports.org/sysutils/xstow/) to manage all 
of my '.dotfiles', this is specific to my personal setup, you can easily integrate your own
'.dotfiles' system. You could also bypass this whole idea and place your jail '.dotfiles' in the 'default-config'
template. That is simple but if you are always tweaking your '.dotfiles' and setup, this allows your latest
changes to be updated across all your jails, keeping your tui consistent across your jails and host.

- I mount my letsencrypt SSL certs and share them with other jails that need them. One jail manages the
certs, the other jails have a read only mount.

- "Living service" configs are saved on a separate datapool /tank/Services. 

For dynamic jails I keep my data on a separate zfs datapool(tank) this separates my data from the
ephemeral jails. If I use Gitea as an example, I have a dataset on my host for the Gitea config and database.
located at '/tank/Services/Gitea'. 

"tree -L 2 /tank/Services/Gitea"
~~~
/tank/Services/Gitea
├── Config
│   └── conf
└── Database
    ├── data
    ├── gitea-repositories
    ├── gitea.db
    └── indexers
~~~

I will mount those directories into my Gitea jail in my Gitea template.

"cat gitea/Bastillefile"
~~~
CMD printf '####\n#### Setup: Mounts and Permissions\n####\n'
CMD mkdir -p /usr/local/etc/gitea
MOUNT /tank/Services/Gitea/Config usr/local/etc/gitea nullfs rw 0 0
CMD mkdir -p /var/db/gitea
MOUNT /tank/Services/Gitea/Database var/db/gitea nullfs rw 0 0
~~~

Now I can rebuild my gitea jail fresh and keep my config and data(Repos) across rebuilds.
I could clone the config and data, create a test gitea jail using the cloned dataset
and experiment with a new version, using a copy of my data. If there are issues, you can just 
destroy the jail and the dataset, continue using your current setup. 

Some service configs are static, we don't worry about creating host mounts
if the jails purpose does not require dynamic data. Such as a jail to monitor
(DDNS)[https://www.cloudflare.com/learning/dns/glossary/dynamic-dns/].

- All of my jails share /var/cache/pkg with the host. This saves space and can speed up rebuilding
jails. If you are testing bastille templates multiple times, any package you recently installed 
on the host or another jail will be cached. This saves internet bandwidth and more importantly, 
rebuild speed!

"cat default-configs/"Bastillefile"
~~~
CMD printf '####\n#### Setup: Mounts and Permissions\n####\n'
# share pkg cache between host/jails
CMD mkdir -p /var/cache/pkg
MOUNT /var/cache/pkg var/cache/pkg nullfs rw 0 0
~~~

# Host settings

I always force resolve.conf to use my routers static IPv4 address and IPv6 unique local address (ULA).
I will copy this file from the host to each of my jails using my 'default-config' template. When using VNET 
jails, this will make sure your jails don't lose IPv6 dns resolving when your IPv6 address is changed by your ISP.
This could possibly be unique to my ISP and network setup, but I don't see any downsides of doing this.
I am from Canada and I know Germans have similar problems with dynamic IPv6 work arounds.

Below is an example /etc/resolvconf.conf, change the IP addresses to something suitable for your setup.

"/etc/resolvconf.conf" 
~~~
name_servers="10.10.10.1 fd00:1234::1"
~~~

# Templates

The templates related to this post are located at: 
[https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips](https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips). The 'default-configs' changes some default FreeBSD settings to reduce cpu workload and writes to disk.

"tree -L 2 default-configs/etc"
~~~
default-configs/etc/
├── newsyslog.conf
├── newsyslog.conf.d
│   ├── ftp.conf
│   ├── lpr.conf
│   ├── opensm.conf
│   ├── pf.conf
│   ├── ppp.conf
│   └── sendmail.conf
├── periodic.conf
├── syslog.conf
└── syslog.d
    ├── ftp.conf
    ├── lpr.conf
    └── ppp.conf
~~~

You may add your own FreeBSD base /etc/ modifications, place your tweaked files in default-configs/etc.
Consider the changes you make here, they could have security implications. Changing syslog.conf as I have
might not be advisable. 
