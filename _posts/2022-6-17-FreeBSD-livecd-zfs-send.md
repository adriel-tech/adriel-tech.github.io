---
layout: post
title: "FreeBSD 13.X - Use live CD & zfs to swap HDD mirror to SSD mirror"
categories: [FreeBSD13, ZFS]
---

WARNING: This is a general guide I made while doing this task, don't follow this blindly.
I hope the general information will help you accomplish a similar task.

This is how I converted a server with a limited number of sata ports from a HDD mirror
to a new SSD ZFS mirror. Using a separate computer and a FreeBSD live CD to create
the new SSD ZFS mirror, transfer the original servers ZFS mirror over the network to the new
SSD ZFS mirror, replace the HDD mirror in the server with the new SSD mirror. Keeping the old
HDD mirror as a physical backup while I am at it.

[1. Spare PC: Install and setup](#1. Spare PC: Install and Setup)

[2. Spare PC: live CD](#2. Spare PC: live CD)

[3. Server](#3. Server)

[4. Finish up Server](#4. Finish up Server)
 
1. Spare PC
  - Use a FreeBSD live CD
    - install onto new ssd mirror

2. Spare PC
  - reboot to live CD. 
    - make /tmp/zroot2
    - mount mirror to zroot2
    - allow root to ssh
    - start sshd

3. Server
  - Stop all services
  - Snapshot zroot
  - ZFS Send zroot to Spare PC
  - Set zfs boot
  - Edit Spare PC /tmp/zroot2/etc/fstab and remove SWAP if required.
  - Power off

4. Server
  - Replace HDD mirror with new SSDs
  - UEFI: change boot to SSD mirror
  - Boot working system 🤞

# 1. Spare PC: Install and Setup

I am going to assume you already know how to do this.
Install your new SSDs into a spare computer, boot from an install disk.
Install Freebsd with a zfs mirror, you don't need to adjust anything else all your
settings will be brought over from your HDD raid on the server.

# 2. Spare PC: live CD

Reboot your fresh install back to the live CD. Make a new temporary directory for the zroot mirror
you created in the previous step, then import it.
~~~
mkdir /tmp/zroot2
zpool import -f -R /tmp/zroot2 zroot
~~~

Setup network, make a unionfs of etc, enable root on ssh, by editing /etc/ssh/sshd_config
and start it.
~~~
dhclient igb0
mkdir /tmp/etc
mount_unionfs /tmp/etc /etc
passwd root
vi /etc/ssh/sshd_config
service sshd onestart
~~~

# 3. Server

Stop all your services or mount the drive mirror in a live CD.
I stopped all my services. Snapshot zroot, send to remote host, set mount point, shutdown
~~~
zfs snapshot -r zroot@mvssd2
zfs send -R zroot@mvssd2 | ssh root@SPAREPCip zfs recv -F zroot
ssh root@SPAREPCip "zpool set bootfs=zroot/ROOT/default zroot"
~~~

!!!
Before powering down: Comment out swap in /tmp/zroot2/etc/fstab because my SSD doesn't
have swap partition. If you kept a swap partition when you installed FreeBSD on the SSDs
you can just poweroff the spare pc.

~~~
ssh root@SPAREPCip "poweroff"
~~~
!!!

# 4. Finish up Server

- Replace HDD mirror in the server with the new SSDs.
- Change UEFI to boot from SSD mirror.
- Boot the system and everything 🚨SHOULD🚨 be the same as it was with the old mirror.
