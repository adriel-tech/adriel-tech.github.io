---
layout: post
title: "FreeBSD 13.X - BastilleBSD Tips"
categories: [BastilleBSD, FreeBSD13]
---

These are a few tricks I use for [BastilleBSD](https://bastillebsd.org/) on FreeBSD 13 to create persistent jails. This document assumes you
have a pc/server on your home lan, you want to use IPv6 which has dynamic addresses and BastilleBSD
is installed and setup. I write this to share my ideas with others familiar with FreeBSD, nothing is to be used without
modification. BastilleBSD can be simple or difficult depending on what you want to do or HOW you do it.
I have found that most examples for BastilleBSD are very basic, that is why I am writing this guide.

[1. Preamble](#preamble)

[2. Host settings](#host-settings)

[3. Templates](#templates)

[4. Building jails](#building-jails)

[5. Conclusion & thoughts](#conclusion-and-thoughts)

---

# Preamble

The flow of my bastille system is as such:

1. Customize a 'default-config' template for a consistent jail setup.
Include 'default-config' template in all of your Bastillefiles. Do not edit
BastilleBSDs default template, it easily causes problems. Our 'default-config' will
overlay BastilleBSDs default settings.

2. Copy our template, modify template, build a jail, apply template.

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
  if the jails purpose does not require persistent data. Such as a jail to monitor
  [DDNS](https://www.cloudflare.com/learning/dns/glossary/dynamic-dns/) or a jail for testing.

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
mkdir -p /usr/local/jails/share
~~~

~~~
git clone https://github.com/adriel-tech/stow-example /usr/local/jails/share/dotfiles
~~~

# Templates

The templates related to this post are located at: 
[https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips](https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips).


## Install templates

~~~
bastille bootstrap https://github.com/adriel-tech/FreeBSD13-BastilleBSD-Tips
~~~

## Modify templates

Lets check out the 'default-configs' template, these settings will apply to all of our jails.
~~~
cd /usr/local/bastille/templates/adriel-tech/FreeBSD13-BastilleBSD-Tips
~~~

"cat default-configs/Bastillefile"
~~~
# Helps with VNET DNS resolving especially ipv6. Without it my VNET jails
# end up with empty resolve.conf on reboot.
CP /etc/resolvconf.conf etc/resolvconf.conf

CMD printf '####\n#### Setup: Mounts and Permissions\n####\n'
# share pkg cache between host/jails
CMD mkdir -p /var/cache/pkg
MOUNT /var/cache/pkg var/cache/pkg nullfs rw 0 0
# Setup home directory
CMD mkdir /root/.config
CMD mkdir /root/.dotfiles
CMD mkdir -p /root/.local/bin
CMD mkdir -p /root/.local/share
# Mount shared dotfiles
MOUNT /usr/local/jails/share/dotfiles root/.dotfiles nullfs ro 0 0

CMD printf '####\n#### Setup: SYSRC\n####\n'
# Tweak bastille's rc.conf
CMD sysrc sendmail_enable="NONE"
CMD dumpdev="NO"

CMD printf '####\n#### Install Programs\n####\n'
# Set PKG to latest, create FreeBSD.conf.bak
CMD sed -i .bak "s/quarterly/latest/g" /etc/pkg/FreeBSD.conf
# Install programs
PKG git-lite htop neovim tree xstow

# Set Jail password, better than nothing
CMD echo "CHANGEME" | pw usermod root -h 0

CMD rm /root/.cshrc
CMD rm /root/.shrc
CMD rm /root/.k5login
CMD rm /root/.login
CMD rm /root/.profile

# Setup configs
CMD xstow -d /root/.dotfiles bin-files
CMD xstow -d /root/.dotfiles git
CMD xstow -d /root/.dotfiles nvim
CMD xstow -d /root/.dotfiles tcsh-root

CMD printf '####\n#### Setup: Overlay etc\n####\n'
OVERLAY etc

CMD printf '####\n#### Nuke default logs\n####\n'
CMD rm -rfv /var/log/*

CMD printf '####\n#### Setup: Post commands\n####\n'
# Date the Jails creation
CMD touch /root/"Created_`date +"%m_%d_%Y"`"
~~~

<p align="center" width="100%">
    <img src="/assets/images/posts/2021-11-12-BastilleBSD-Tips/what-is-this.gif"> 
</p>

What did we see?
- Hosts resolvconf.conf copied to jail.
- Mount host /var/cache/pkg.
- Root user home directory setup + dots #1.
- Sysrc tweaks.
- Pkg set to use latest instead of quarterly (latest packages).
- Install must have programs: "git-lite htop neovim tree xstow".
- Give root a default password "CHANGEME".
- Root user home directory setup + dots #2.
- Overlay my custom /etc/ files to overwrite FreeBSD defaults.
- Nuke the logs.
- Create a file in /root so I know the creation date of the jail.

My 'default-configs' template changes some default FreeBSD settings to reduce cpu workload and writes to disk.

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

Take a look through default-configs and add/remove things to your hearts content. Every template
you create from this point forward will include default-configs. If you are not using xstow or dotfiles, you 
can delete that section from the Bastillefile.

~~~
# Setup home directory
CMD mkdir /root/.config
CMD mkdir /root/.dotfiles
CMD mkdir -p /root/.local/bin
CMD mkdir -p /root/.local/share
# Mount shared dotfiles
MOUNT /usr/local/jails/share/dotfiles root/.dotfiles nullfs ro 0 0

# Setup configs
CMD xstow -d /root/.dotfiles bin-files
CMD xstow -d /root/.dotfiles git
CMD xstow -d /root/.dotfiles nvim
CMD xstow -d /root/.dotfiles tcsh-root
~~~

## Using default-configs in another template

Let's make a new Bastille file for nginx, goaccess and letsencrypt to see what we can do. I created a template with 
the layout I like for Bastillefiles, we'll copy it and modify it.

~~~
cp -R _template/ nginx-TEST-setup
~~~

'cat nginx-TEST-setup/Bastillefile'
~~~
CMD printf '####\n#### Setup: Defaults\n####\n'
INCLUDE /usr/local/bastille/templates/adriel-tech/FreeBSD13-BastilleBSD-Tips/default-configs

CMD printf '####\n#### Setup: Mounts and Permissions\n####\n'


CMD printf '####\n#### Setup: SYSRC\n####\n'


CMD printf '####\n#### Install Programs\n####\n'


CMD printf '####\n#### Setup: Overlay etc\n####\n'


CMD printf '####\n#### Setup: Pre start commands\n####\n'


CMD printf '####\n#### Start Services\n####\n'


CMD printf '####\n#### Setup: Post commands\n####\n'
~~~

Now let's build up our nginx-TEST-setup Bastillefile. I want this jail to be a VNET jail with IPv6 and a
default PF firewall enabled. I include the 'default-configs' of course, also 2 'snippet' Bastillefiles
that will enable ipv6 and pf with a basic pf.conf. 

~~~
CMD printf '####\n#### Setup: Defaults\n####\n'
INCLUDE /usr/local/bastille/templates/adriel-tech/FreeBSD13-BastilleBSD-Tips/default-configs
INCLUDE /usr/local/bastille/templates/adriel-tech/FreeBSD13-BastilleBSD-Tips/snippet_ipv6
INCLUDE /usr/local/bastille/templates/adriel-tech/FreeBSD13-BastilleBSD-Tips/snippet_pf

CMD printf '####\n#### Setup: Mounts and Permissions\n####\n'
CMD mkdir -p /usr/local/etc/letsencrypt
MOUNT /usr/local/jails/share/ule-letsencrypt usr/local/etc/letsencrypt nullfs ro 0 0
CMD mkdir -p /usr/local/etc/www
MOUNT /usr/local/jails/nginx-TEST/ul-www usr/local/www nullfs rw 0 0
CMD mkdir -p /usr/local/etc/nginx
MOUNT /usr/local/jails/nginx-TEST/ule-nginx usr/local/etc/nginx nullfs rw 0 0

## Init goaccess
CMD mkdir -p /usr/local/www/goaccess
CMD touch /usr/local/www/goaccess/index.html
CMD chmod 644 /usr/local/www/goaccess/index.html

CMD printf '####\n#### Setup: SYSRC\n####\n'
SYSRC goaccess_enable="YES"
SYSRC nginx_enable="YES"

CMD printf '####\n#### Install Programs\n####\n'
PKG goaccess nginx-devel py39-certbot

#CMD printf '####\n#### Setup: Overlay etc\n####\n'
#OVERLAY etc

CMD printf '####\n#### Setup: Pre start commands\n####\n'
CMD nginx -t

CMD printf '####\n#### Start Services\n####\n'
SERVICE nginx start

#CMD printf '####\n#### Setup: Post commands\n####\n'
~~~

We are almost ready to build this jail up and modify it to our liking.
Before we do that, on the host we will make the directories needed for our 
persistent data; letsencrypt, www, nginx config.
~~~
mkdir -p /usr/local/jails/share/ule-letsencrypt
mkdir -p /usr/local/jails/nginx-TEST/ul-www
mkdir -p /usr/local/jails/nginx-TEST/ule-nginx
~~~

# Building jails

## nginx-TEST-setup

Create the jail with Bastille, then activate the templates. This is considered a new project that needs
some other setup. Once the jail is built and setup we will pop inside, customize what we want and play a bit.

Create VNET jail:
~~~
bastille create -V nginx-TEST-setup 13.1-RELEASE 10.10.10.42 em0
~~~

Apply template to new jail:
~~~
bastille template nginx-TEST-setup adriel-tech/FreeBSD13-BastilleBSD-Tips/nginx-TEST-setup
~~~

Assuming there are no problems, we can enter the jail and play around.
Bastille does not handle errors at all really. If I have problems with the template propagating, I
read over the error, try to fix it, then nuke the jail and rebuild it, reapplying a template is not
something you should do.

~~~
bastille console nginx-TEST-setup
~~~

At this point I will get letsencrypt running, tweak up my nginx config, modifiy my /etc/pf.conf to my liking.
Assuming I am happy with it all, I will decide what configs can stay in my template and which I want on the host.
I made this decision earlier since I have done this already (letsencrypt, www, nginx).

Exit out of the jail and now we on the host. I have a basic nginx template, I'll make a copy of it
and add in my working configs from the current jail.

## nginx-TEST-final

~~~
cp -R nginx-TEST-setup/ nginx-TEST-final
~~~

create folders for static configs
~~~
mkdir nginx-TEST-final/etc
mkdir -p nginx-TEST-final/usr/local/etc
~~~

Copy modded nginx-TEST-setup /etc/pf.conf. goaccess config is not something I need to mess with
often, it can be copied into the template also.
~~~
cp /usr/local/bastille/jails/nginx-TEST-setup/root/etc/pf.conf nginx-TEST-final/etc/
cp -R /usr/local/bastille/jails/nginx-TEST-setup/root/usr/local/etc/goaccess/ nginx-TEST-final/usr/local/etc/goaccess
~~~

Edit the README in nginx-TEST-final with any info/notes you want too. Now lets make a new jail and try out the new
config, we made no mistakes and this is all going to work the first time!

Create VNET jail:
~~~
bastille create -V nginx-TEST-final 13.1-RELEASE 10.10.10.43 em0
~~~

Apply template to new jail:
~~~
bastille template nginx-TEST-final adriel-tech/FreeBSD13-BastilleBSD-Tips/nginx-TEST-final
~~~

If your template fails, you'll need to troubleshoot it, stop, destroy jail then start again.
You kept the original nginx-TEST-setup jail, you can also compare your work with it.
Assuming the template works without issue, you are done.

## Update Jails

You can update the jail in the future from the host by running:
~~~
bastille cmd nginx-TEST-final pkg upgrade -y
~~~

You can also rebuild the jail, our letsencrypt , nginx and www files will mount back in:
~~~
bastille stop nginx
bastille destroy nginx
bastille create -V nginx-TEST-final 13.1-RELEASE 10.10.10.43 em0
bastille template nginx-TEST-final adriel-tech/FreeBSD13-BastilleBSD-Tips/nginx-TEST-final
~~~

# Conclusion and thoughts

I don't usually create 2 templates '-setup' and '-final', I used them as a tool to explain things easier.
Integrate your own snippets for simple generic things to save time. You could disable syslogd and cron for your jails,
which makes sense for some and not others. Make a snippet and add it to the jails that don't need it. Once you
create your own defaults and build up 1-2 services using what you learned from this post, you'll
understand the power of it.

I hope you learned from this post and generated some ideas that will improve your
own setup, please share them with the community!