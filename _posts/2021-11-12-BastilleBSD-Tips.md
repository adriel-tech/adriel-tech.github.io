---
layout: post
title: "FreeBSD 13 BastilleBSD Tips"
categories: [BastilleBSD, FreeBSD13]
---

These are a few tricks I use for [BastilleBSD](https://bastillebsd.org/) on FreeBSD 13. This document assumes you
have a home server on your lan, you want to use IPv6, your IPv4 and IPv6 are both dynamic addresses. You have
installed BastilleBSD and are using ZFS. This setup can easily be used for public servers or laptops, use my
examples to improve your own setup. IPv6 is easy to disable if you don't need it.

[1. Preamble](#preamble)

[2. Host settings](#host-settings)

[3. Templates](#templates)

[4. Building jails](#building-jails)

---

# Preamble

The flow of my bastille system is as such:

1. Dial in the default-config template for a consistent base jail setup.
Install your important programs and settings so everything is as you like.

2. Copy a template, modify template, build a jail, apply template.

3. Enter new jail, fine tune the setup and modify it to your liking.

4. Once everything is good, copy the configs out of the jail and into your
template. Rebuild jail with new configs, did everything work? Good we are done.

5. Future tweaks and updates can be ported back to the Bastille template and jail can be easily
rebuilt or cloned. I like using jails this way, I can pop in a live jail, tweak and test it, then
update the template.

Other stuff:

- I mount my '.dotfiles' for jails in a shared folder. This allows a simple way to update userland
configs across all jails. I use [Xstow](https://www.freshports.org/sysutils/xstow/) to manage all 
of my '.dotfiles', this is specific to my personal setup, you can easily integrate your own
'.dotfiles' system. You could also bypass this whole idea and place your jail '.dotfiles' in the 'default-config'
template. That is simple but if you are always tweaking your '.dotfiles' and setup, this allows your latest
changes to be updated across all your jails, keeping your tui consistent across your jails and host.

- I mount my letsencrypt SSL certs and share them with other jails that need them. One jail manages the
certs, the other jails have a read only mount.

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

- For dynamic jails I keep my data on a separate zfs datapool(tank) this separates my data from the
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
and experiment with a new version, using a copy of my data. If there are issues, just
destroy the jail and the dataset, continue using your current setup.

Some service configs are static, we don't worry about creating host mounts
if the jails purpose does not require dynamic data. Such as a jail to monitor
[DDNS](https://www.cloudflare.com/learning/dns/glossary/dynamic-dns/).

# Host settings

## /etc/resolveconf.conf

I always force resolve.conf to use my routers static IPv4 address and IPv6 unique local address (ULA).
I will copy this file from the host to each of my jails using my 'default-config' template. When using VNET
jails, this will make sure your jails don't lose IPv6 dns resolving when your IPv6 address is changed by your ISP.
This could possibly be unique to my ISP and network setup, but I don't see any downsides of doing this.
I am from Canada and I know Germans have similar problems with dynamic IPv6 workarounds. Below is an example
/etc/resolvconf.conf, change the IP addresses to something suitable for your setup.

"cat /etc/resolvconf.conf" 
~~~
name_servers="10.10.10.1 fd00:1234::1"
~~~

## Dotfiles

~~~
zfs create -p zroot/usr/local/jails/share
~~~

~~~
git clone https://github.com/YOURDOTS/.dotfiles.git /zroot/usr/local/jails/share/dotfiles
~~~

# Templates

The templates related to this post are located at: 
[https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips](https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips).
The 'default-configs' template changes some default FreeBSD settings to reduce cpu workload and writes to disk.

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

## Install templates

~~~
git clone https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips /usr/local/bastille/templates/FreeBSD13-BastilleBSD-Tips
~~~

~~~
cd /usr/local/bastille/templates/FreeBSD13-BastilleBSD-Tips
~~~

Take a look through default-configs, Bastillefile and /etc folder and tweak to your liking, if you are not
using xstow or dotfiles, you can delete that section from the Bastillefile.

~~~
# Setup home directory
CMD mkdir /root/.config
CMD mkdir /root/.dotfiles
CMD mkdir -p /root/.local/bin
CMD mkdir -p /root/.local/share
# Mount shared dotfiles
MOUNT /zroot/usr/local/jails/share/dotfiles root/.dotfiles nullfs ro 0 0

# Setup configs
CMD xstow -d /root/.dotfiles bin-files
CMD xstow -d /root/.dotfiles cheat
CMD xstow -d /root/.dotfiles git
CMD xstow -d /root/.dotfiles nvim
CMD xstow -d /root/.dotfiles tcsh-root
~~~

## Using default-configs

default configs will be included in every Bastillefile we create from now on. Let's make a new Bastille file
for nginx. I have a template with the layout I like for Bastillefiles, we'll copy it and modify it.

~~~
cp -R _template/ nginx-TEST
~~~

'cat nginx-TEST/Bastillefile'
~~~
CMD printf '####\n#### Setup: Defaults\n####\n'
INCLUDE /usr/local/bastille/templates/FreeBSD13-BastilleBSD-Tips/default-configs

CMD printf '####\n#### Setup: Mounts and Permissions\n####\n'


CMD printf '####\n#### Setup: SYSRC\n####\n'


CMD printf '####\n#### Install Programs\n####\n'


CMD printf '####\n#### Setup: Overlay etc\n####\n'


CMD printf '####\n#### Setup: Pre start commands\n####\n'


CMD printf '####\n#### Start Services\n####\n'


CMD printf '####\n#### Setup: Post commands\n####\n'
~~~

Now we will build up or nginx Bastillefile. I want this jail to be a VNET jail
with IPv6 and a default PF firewall enabled. I am including 2 'snippet' Bastillefiles
that will enable ipv6 and enable pf with a basic pf.conf. On the host I make a folder to
store my letsencrypt certs on the host, so I can share the with other jails. 
'mkdir -p /zroot/usr/local/jails/share/ule-letsencrypt'. I will boot this jail up
modify it to my liking, then I will probably remove, 'snippet_pf' from this include
and copy my customized pf.conf out of the jail and into this nginx-TEST/etc folder
to be overlayed into my final template. 

~~~
CMD printf '####\n#### Setup: Defaults\n####\n'
INCLUDE /usr/local/bastille/templates/FreeBSD13-BastilleBSD-Tips/default-configs
INCLUDE /usr/local/bastille/templates/FreeBSD13-BastilleBSD-Tips/snippet_ipv6
INCLUDE /usr/local/bastille/templates/FreeBSD13-BastilleBSD-Tips/snippet_pf

CMD printf '####\n#### Setup: Mounts and Permissions\n####\n'
CMD mkdir -p /usr/local/etc/letsencrypt
MOUNT /zroot/usr/local/jails/share/ule-letsencrypt usr/local/etc/letsencrypt nullfs ro 0 0

CMD printf '####\n#### Setup: SYSRC\n####\n'
SYSRC goaccess_enable="YES"
SYSRC nginx_enable="YES"

CMD printf '####\n#### Install Programs\n####\n'
PKG goaccess nginx-devel

CMD printf '####\n#### Setup: Overlay etc\n####\n'
OVERLAY etc

CMD printf '####\n#### Setup: Pre start commands\n####\n'
CMD nginx -t

CMD printf '####\n#### Start Services\n####\n'
SERVICE nginx start

CMD printf '####\n#### Setup: Post commands\n####\n'
~~~

# Building jails

We are going to build out a jail now, we'll create the jail with Bastille, then activate the templates.
This is considered a new project that needs some other setup. Once the jail is built and setup
we will pop inside and customize what we want, play with it  a bit. Once we are happy, we can copy 
any specific config changes out of the jail and into our nginx-TEST/usr/local/etc folder.
We can then build a new jail with the modified configs already setup. Assuming we got everything working
then this template is completed. I use my jails as work spaces also and port my changes back into
my templates, and document them in the README in each template. This way I can view them on gitea/github,
documenting your Bastillefile and putting notes in a README are advised.

Create VNET jail:
~~~
bastille create -V nginx-TEST 13.1-RELEASE 10.10.10.42 em0
~~~

Apply template to new jail:
~~~
bastille template nginx-TEST FreeBSD13-BastilleBSD-Tips/nginx-TEST
~~~

Assuming there are no problems, we can enter the jail and play around.
Bastille does not handle errors at all really. If I have problems with the template propagating, I
read over the error, try to fix it, then nuke the jail and rebuild it, reapplying a template is not
something you should do.

~~~
bastille console nginx-TEST
~~~
