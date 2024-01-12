---
layout: post
title: "FreeBSD - Zerotier mesh network with jail support"
categories: [FreeBSD]
---

[Zerotier](https://www.zerotier.com/) is a mesh vpn network which allows you to connect
your computers/servers/devices in a private network. It is incredibly useful and really cool technology.
Installing and using it is simple enough, it even works inside jails with a few extra steps.

## Make a Zerotier account

Go to [https://www.zerotier.com/](https://www.zerotier.com/) and create an account.
The next step is to create a network, you will need the network ID to join
your FreeBSD server to the network later on.

## Install Zerotier

~~~
pkg install -y zerotier
~~~

~~~
sysctl net.link.tap.up_on_open=1
~~~

~~~
echo "net.link.tap.up_on_open=1" >> /etc/sysctl.conf
~~~

## Enable Service

~~~
sysrc zerotier_enable="YES"
~~~

~~~
service zerotier start
~~~

## Join Network

~~~
zerotier-cli join YOURnetworkID
~~~

## Accept device from webmin

Go back to [https://www.zerotier.com/](https://www.zerotier.com/) and you should see
your device in the network. You need to allow it from the webui and then it is connected.

## Allow Zerotier in jails

Zerotier can only work in VNET jails which were enabled by default sometime in FreeBSD 12.
On your FreeBSD host you will need to add some devfs rules to allow the jails to interact with
TAP devices. Zerotier uses a TAP device and then renames it to your network ID. Without these
devfs rules the jail will not be able to interact with TAP devices and zerotier will not work.

edit /etc/devfs.rules add the below rule

~~~
# Added for zerotier VNET jails, tap for zerotier
[zerotier_vnet=14]
add include $devfsrules_hide_all
add include $devfsrules_unhide_basic
add include $devfsrules_unhide_login
add include $devfsrules_jail
add include $devfsrules_jail_vnet
add path 'bpf*' unhide
add path 'tap*' unhide
~~~

Now you need to set your jail to use the newly created devfs rule. This depends on what type of
jail management system you are using. I will pretend you are using a generic jail.conf.

edit jail.conf

~~~
zeroter-jail {
  devfs_ruleset = 14;
  ...
  ...
  ...
}
~~~

Once this is done you can launch your jail and follow the steps from the start of this guide
to connect the jail to your zerotier network as a new seperate device.

