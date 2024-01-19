---
layout: post
title: "FreeBSD 14 - Remove deprecated /usr/home symlink"
categories: [FreeBSD14]
---

Freebsd 14 was released with [this commit.](https://cgit.freebsd.org/src/commit/?id=bbb2d2ce4220){:target="_blank"}
In the past users home directories were placed in /usr/home with a symlink to /home.
Freebsd 14+ will return to using /home directly, this is great. Over the years I have
bumped into minor issues with programs due to the /usr/home diretory, [broot](https://github.com/Canop/broot){:target="_blank"}
being a recent one.

If you have upgraded from an earlier version of FreeBSD you will still be using the old symlink. Below
I will show you how to remove it. All of these commands assume you are using root, add sudo/doas to them
if you need to.

## ZFS instructions

- Remove the /home symlink
- create zroot/home dataset and mount it to /home
- move your user folder to the new dataset
- destroy the old zroot/usr/home dataset

~~~
rm /home
zfs create -o mountpoint=/home zroot/home
mv /usr/home/adriel-tech/ /home/adriel-tech
zfs destroy zroot/usr/home
~~~

## UFS instructions

- Remove the /home symlink
- create /home folder
- move your user folder to the new folder
- delete the old and empty /usr/home folder

~~~
rm /home
mkdir /home
mv /usr/home/adriel-tech/ /home/adriel-tech
rmdir /usr/home
~~~
