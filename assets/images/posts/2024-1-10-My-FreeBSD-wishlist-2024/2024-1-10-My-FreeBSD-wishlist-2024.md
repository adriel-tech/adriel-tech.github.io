---
layout: post
title: "My FreeBSD wish list 2024 edition"
categories: [FreeBSD] [musing]
---

This is a list of things I'd like to see in FreeBSDs future.

<p align="center" width="100%">
    <img src="/assets/images/posts/2024-1-10-My-FreeBSD-wishlist-2024/wishlist.jpg"> 
</p>

## Thin out the base system.

### Remove all user shells but the default sh

This reduces code to maintain without adding much stress to users. This makes more sense now that FreeBSD 14
switched the [default user shell](https://lists.freebsd.org/archives/freebsd-current/2021-September/000648.html) to /bin/sh.
It is common to install your non standard preferred user shell on most Unix likes and takes minimal effort.

~~~
pkg install -y bash fish tcsh zsh
~~~

### Remove IPFILTER (IPF)

I've used FreeBSD for 16 years and I have never seen a guide or user seriously suggest using
IPF. PF and IPFW are heavily used, they each have benefits and actively receive features.
This reduces code to maintain and we still have multiple viable alternates in the base system.

### Blah

## Better defaults for userland.

FreeBSD is very conservative about making changes. The suggestions below could be added
as a simple end of installer question, "Use old FreeBSD shell defaults or modern defaults?".

### Shell

- Put the classic FreeBSD up/down arrow history navigation from tcsh and put in
our default shell configuration.

- Update the prompt with extra features & color.

- Turn color on everything that supports it by default, shell prompt, ls, grep, ???.
  - I am talking about minimal useful color, not RGB rainbow output.

I thought I seen FreeBSD support for [NO_COLOR](https://no-color.org), I could be wrong.
But 'ls -G' and 'grep --color=auto' both suppress the color when piped. It would be ideal
to make it easy for users to opt out of color with the environment variable 'NO_COLOR', it would be
easy enough to add to the default shell script, either way.

### Pkg

- Add a dash of color and a nice looking progress bar, modern APT on Debian has this and
it looks pretty good without being too shiny or intrusive.

<p align="center" width="100%">
    <img src="/assets/images/posts/2024-1-10-My-FreeBSD-wishlist-2024/apt.png"> 
</p>

### ifconfig

ifconfig is a beast and I can understand why nobody wants to dig into it to add extra features that increase complexity.
This is a wish list though and so we are going to do some wishing.

- Add color like Linux 'ip addr' command.
- a way to reduce some of the information
- default to non hex netmask....

~~~
ifconfig show mac ip
~~~

~~~
em0: ether ec:b1:e3:10:8g:71
     inet 172.16.2.100 netmask 255.255.255.0 broadcast 172.16.2.255
     inet6 fe80::100 prefixlen 64 scopeid 0x7

bridge0: ether ec:b1:e3:10:8g:72
     inet 10.0.1.10 netmask 255.255.255.255
     inet 10.0.1.11 netmask 255.255.255.255
     inet 10.0.1.12 netmask 255.255.255.255
     inet6 fe80::1%bridge0 prefixlen 64 scopeid 0x5

tap0: ether ec:b1:e3:10:8g:73
~~~

The above is just an example. ifconfig has a lot of info especially if you have a few interfaces, jails
and virtual machines. You can end up with 2-3 screens worth of information.

### PF default pre configured rules.

IPFW has some built in basic configurations such as workstation 'sysrc firewall_type="workstation"'.
It would be nice to have similar for PF even if its something mentioned in the [Handbook](https://docs.freebsd.org/en/books/handbook/firewalls)
and users could copy it over to /etc/pf.conf.

This would give people a basic default structure to build from. FreeBSD is making strides at becoming a Tier 1
[Cloud-init Platform](https://freebsdfoundation.org/project/freebsd-as-a-tier-i-cloud-init-platform),
having a quick firewall default that can be copied over or set at install time would be a benefit.

The installer could have a firewall section:
"Enable a firewall? Yes/No
"Which firewall? IPFW/PF
"What default configuration?" workstation/cloud

workstation would be block all in and let all out.
cloud would be block all in, let ssh through.

Obviously this needs some more thought to make it easy, quick and understanble for users.
We can't make everyone happy, that does not mean we can't have a few basic options to get people started.
They can have a PF firewall auto setup for their cloud install and look through the handbook to learn how 
to add the other firewall features they need.

## Jails

I love jails, I used many different jail managers in the past. I did not really see their potential until
I worked through [Freebsd Jails the hard way](https://clinta.github.io/freebsd-jails-the-hard-way). I
learned from this experience that jails can be as complicated as your poor FreeBSD knowledge allows them to be.
I later moved to [BastilleBSD](https://github.com/BastilleBSD/bastille) for it's automation. So what is there
to improve?

^^^ Need tweaks. ^^^
prob nuke and flow into below ^^^

### Default jail userland tools

FreeBSD gives you access to a lot of tools but the base system does not give you structured interface to use it.
The benefit of this is your imagination is the limit, but some of us don't have great imaginations. So we use one of many
jail management systems and everyone uses something different. Having options is great but it would be nice to have and official
jail management tool in the base system.

Imagine jailctl (we already have bhyvectl), with a few simple commands we end up with a nice working
jail.conf.

====
more here
====

### Thin images

This can also be generalized to virtual machines.
Thin reduced images meant to have the most basic tools like many docker images do.
A common use for jails is program separation, It would be nice to have a thin image
that defaults to everything off if not reduced. If I am using a jail to containerize 
a [Mumble voice chat server](https://www.mumble.info), I don't need cron, periodic, newsyslog, syslog
or sendmail. Mumble has a single config file and it does it's own logging to a single file.
I use automated scripts and such to disable most of this, but this is a common use case for
containers including jails and having to rip things out and disable parts of your container
is more annoying than adding what you need to a clean slate.

I guess [pkgbase](https://wiki.freebsd.org/PkgBase) will help with this? I am thinking
more in the direction of a container image 'FreeBSD-14-mini-contaier.img' that is stripped down
similar to other images FreeBSD already makes. Jails are powerful and plenty of times I want a full
FreeBSD install in a VNET jail, this is idea is just an option for other use cases.

## Virtual machines (bhyve)

I basically want something like NULLFS for jails but for bhyve. I assume [virtio-fs](https://virtio-fs.gitlab.io) support is the way to get that.

Bhyve currently supports virtio-9p which is [very useful.](https://adriel-tech.github.io/bhyve/nixos/vm/2022/10/11/FreeBSD-13-bhyve-share-files-with-vm.html)
It has limitations that virtio-fs does not have and I think would be useful for FreeBSD.
I want all my files on FreeBSD using ZFS, compression, snapshots and backups.
I want to easily use windows or Linux virtual machines to host apps that FreeBSD cannot but store their
data on the host system without issue or slowdown. I already use NixOS/Debian to run podman/docker and store
the files on FreeBSD, but this is not possible for databases and other types of files.

It is not to say you cannot do this on FreeBSD now, you need to do a lot of work setting up NFS and jumping through hoops and
network connections. I want the ease of sharing files in virtual machines like I have with jails using nullfs.

### Virtual machine thin images

Same as above for jails, you want minimal FreeBSD virtual machines for isolated tasks. 
Bhyve can be used like [AWS Firecracker](https://aws.amazon.com/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/).
Firecracker is a lot more than just a thin image obviously, I am just musing about this.
