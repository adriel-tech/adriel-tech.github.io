---
layout: post
title: "My FreeBSD wish list 2024 edition"
categories: [FreeBSD, musing]
---

This is a list of things I would like to see in FreeBSDs future.

<p align="center" width="100%">
    <img src="/assets/images/posts/2024-1-10-My-FreeBSD-wishlist-2024/wishlist.jpg" alt="Santa upset about FreeBSD changes meme"> 
</p>

## 1. Thin out the base system:

### Remove all user shells but the default shell

This reduces code to maintain without adding much stress to users. This makes more sense now that FreeBSD 14
switched the [default user shell](https://lists.freebsd.org/archives/freebsd-current/2021-September/000648.html){:target="_blank"} to /bin/sh.
It is common to install your personally preferred shell on most unix-like operating systems and takes minimal effort.

~~~
pkg install -y bash fish tcsh zsh
~~~

### Remove IPFILTER (IPF)

I've used FreeBSD for 16 years and I have never seen a guide or user seriously suggest using
IPF. PF and IPFW are heavily used, they each have benefits and actively receive features.
This reduces code to maintain and we still have multiple viable alternates in the base system.

### Replace Ntpd with OpenNTPD

[OpenNTPD](https://www.openntpd.org){:target="_blank"} is smaller and easier to configure, if you need a full blown ntpd server install it from pkg.
I think this lines up with changes of a similar type in [FreeBSD 14.](https://lists.freebsd.org/archives/freebsd-questions/2023-November/004322.html){:target="_blank"}

## 2. "Improved" userland defaults:

FreeBSD is very conservative about making changes. The suggestions below could be added
as question during adding a user, "Use old FreeBSD shell defaults or the new hip defaults?".

### Shell

- Add back FreeBSDs classic up/down arrow history navigation from tcsh to our default shell configuration.

- Update the prompt with extra features & color.

- Turn color on everything that supports it by default: shell prompt, ls, grep, ???.
  - I am talking about minimal useful color, not RGB rainbow output.

I thought I seen FreeBSD support for [NO_COLOR](https://no-color.org){:target="_blank"} in the past, I could be wrong.
It would be ideal to make it easy for users to opt out of color with the environment variable 'NO_COLOR', it would be
easy enough to add to the default shell script. Either way 'ls -G' and 'grep --color=auto' both suppress the color when piped.

### Pkg

- Add a dash of color and a nice looking progress bar, modern APT on Debian has this and
it looks pretty good without being too shiny or intrusive.

<p align="center" width="100%">
    <img src="/assets/images/posts/2024-1-10-My-FreeBSD-wishlist-2024/apt.png" alt="Picture of apt using color."> 
</p>

### ifconfig

ifconfig is a beast and I can understand why nobody wants to dig into it and add extra features that increase complexity.
This is a wish list though and so we are going to do some wishing.

<p align="center" width="100%">
    <img src="/assets/images/posts/2024-1-10-My-FreeBSD-wishlist-2024/wishbutton.jpg" alt="Picture of pushing a wish button."> 
</p>

- Add color similar to the Linux command 'ip addr'.

<p align="center" width="100%">
    <img src="/assets/images/posts/2024-1-10-My-FreeBSD-wishlist-2024/ip-addr.png" alt="Picture of ip using color."> 
</p>

- Add a sub command to choose the information to be shown.
- Default to non hex netmask.... 

~~~
ifconfig show mac ip
~~~

~~~
em0: 
  ether ec:b1:e3:10:8g:71
  inet 172.16.2.100 netmask 255.255.255.0 broadcast 172.16.2.255
  inet6 fe80::100 prefixlen 64 scopeid 0x7

bridge0: 
  ether ec:b1:e3:10:8g:72
  inet 10.0.1.10 netmask 255.255.255.255
  inet 10.0.1.11 netmask 255.255.255.255
  inet 10.0.1.12 netmask 255.255.255.255
  inet6 fe80::1%bridge0 prefixlen 64 scopeid 0x5

tap0:
  ether ec:b1:e3:10:8g:73
~~~

The above is just an example. ifconfig is information dense, if you have a few interfaces, jails
and virtual machines, you can end up with 2-3 screens worth of information.

### PF default pre configured rules.

IPFW has some built in basic configurations such as workstation 'sysrc firewall_type="workstation"'.
It would be nice to have similar for PF even if its something mentioned in the [Handbook](https://docs.freebsd.org/en/books/handbook/firewalls){:target="_blank"}
and users could copy it from /etc/defaults/pf/ to /etc/pf.conf.

This would give people a basic default structure to build from. FreeBSD is making strides at becoming a Tier 1
[Cloud-init Platform](https://freebsdfoundation.org/project/freebsd-as-a-tier-i-cloud-init-platform){:target="_blank"},
having a quick firewall default that can be copied over or set at install time would be a benefit.

The installer could have a firewall section:

~~~
"Enable a firewall?"          Yes/No
"Which firewall?"             IPFW/PF
"What default configuration?" Workstation/Cloud
~~~

Workstation would be block all in and let all out.
Cloud would be block all in, let ssh through.

Obviously this needs more effort and thought to make it easy, quick and understanble for users.
We can't make everyone happy, that does not mean we can't have a few basic default options to get people started.
Users can have a basic PF firewall auto setup for their cloud install and look through the handbook to learn how 
to add more firewall features if needed.

## 3. Jails:

### Default jail userland tools

Jails are an example of a powerful tool in FreeBSD which is built into the base OS and can be used as is.
The problem is that you need to know what buttons to push, where the buttons are and have an understanding of FreeBSD.
That is great but it would be a really nice to have an official jail management tool in the base system.
Users could install and use their favorite jail manager but we should have a more user friendly interface
to access what the operating system already allows us to do. I know above I started with reducing the base OS but
one of the selling points of FreeBSD is the complete OS being a comprehensive kernel and user land built together.
Tighter user land intigration with cool kernel features is something we want across the board.

jailctl?

[@Callfortesting](https://www.youtube.com/@callfortesting){:target="_blank"} have discussed this and they
have a clearer understanding and expectation of what should be done. I hope some of their ideas make it into FreeBSD.

### Thin images

This can also be generalized to virtual machines.
Thin reduced images meant to have the most basic tools like many docker images do.
A common use for jails is program and dependency separation, It would be nice to have a thin image
that defaults to everything off if not reduced. If I am using a jail to containerize 
a [Mumble voice chat server](https://www.mumble.info){:target="_blank"}, I don't need cron, periodic, newsyslog, syslog
or sendmail. Mumble has a single config file and it does it's own logging to a single file.
I use automated scripts and such to disable most of this, but this is a common use case for
containers including jails and having to rip things out and disable parts of your container
is more annoying than adding what you need to a clean slate.

I guess [pkgbase](https://wiki.freebsd.org/PkgBase){:target="_blank"} will help with this? I am thinking
more in the direction of a container image 'FreeBSD-14-mini-contaier.img' that is stripped down
similar to other images FreeBSD already makes. Jails are powerful and plenty of times I want a full
FreeBSD install in a VNET jail, this idea is an option for other use cases.

## 4. Virtual machines (bhyve):

### Default virtual machine userland tools

Similar to jails, having something like [vm-bhyve](https://github.com/churchers/vm-bhyve){:target="_blank"} in the base system with a
similar interface to the new jailctl system we added above. I have manually used bhyve in the past, it works but it would be preferable
to have a more user friendly interface. Users could still install and use their favorite VM manager.

### More robust file sharing with host

I want something like NULLFS for jails but for bhyve, I assume [virtio-fs](https://virtio-fs.gitlab.io){:target="_blank"} support is the way to get that.
Bhyve currently supports virtio-9p which is [very useful.](https://adriel-tech.github.io/bhyve/nixos/vm/2022/10/11/FreeBSD-13-bhyve-share-files-with-vm.html){:target="_blank"}
It has limitations that virtio-fs does not have and I think would be useful for FreeBSD. It would be wonderful to to easily host Windows or Linux
virtual machines and store their important data on the hosts ZFS filesystem without issue or slowdown. I already use NixOS/Debian to run podman/docker
and store the files on FreeBSD, but this is not possible for databases and other types of files.

In general FreeBSD can currently do this, you need to do a lot of work setting up NFS and jumping through hoops and
network connections. I want the ease of sharing files in virtual machines with the host like jails using nullfs.
