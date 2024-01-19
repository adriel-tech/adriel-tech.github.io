---
layout: post
title: "FreeBSD 13 - Bhyve share host files with VMs"
categories: [Bhyve, Bhyve, NixOS, VM]
---

This is how I share files from a FreeBSD 13+ host into a bhyve VM. I host all my files
on my FreeBSD host and pass certain folders into my NixOS VM for podman(docker) to use.
These examples use NixOS but this works for other Linux distributions. I do not know how 
to do this for Windows, you are on your own 🫡. 

## On your FreeBSD host

I use [vm-bhyve](https://github.com/churchers/vm-bhyve){:target="_blank"} to manage my VMs but
it is not required for this to work, we are manually adding our shares without vm-bhyves
help.

### Create your shared folder(s)

~~~
mkdir /mnt/vm-share
mkdir /mnt/vm-share2
~~~

### Edit your VMs bhyve config

I will be editing /usr/local/vm/nixos/nixos.conf. We are looking for
"bhyve_options" we are going to ADD two slots, one for each shared folder.
Do not remove your other bhyve_options or your VM will be broken.

~~~
bhyve_options="-s 15,virtio-9p,vm-share=/mnt/vm-share -s 16,virtio-9p,vm-share2=/mnt/vm-share2"
~~~

Note that each share needs -s # and they must be different, I start at 15 and keep going up from 
there for each folder I am sharing.

### Inside your VM

I am using NixOS, I can't remember the normal command to do this, I'll try and add it in the future.

In your /etc/nixos/configuration.nix

~~~
  fileSystems."/mnt/vm-share" = {
      device = "vm-share";
      fsType = "9p";
      options = [ "trans=virtio" "version=9p2000.L" "_netdev" "cache=loose" ];
  };

  fileSystems."/mnt/vm-share2" = {
      device = "vm-share2";
      fsType = "9p";
      options = [ "trans=virtio" "version=9p2000.L" "_netdev" "cache=loose" ];
  };
~~~

Now do a nixos-rebuild and POWEROFF your VM.

## Finishing up

The 9p mounts won't get passed into the VM until you STOP the VM and then START the VM (restart won't work).
Once that is done and you can confirm the files are passed through and working, things 'just work'. 
This does not work for things that require file locks like passing database files through 9p.

To test things you can add a file from the host, boot up your VM and check if it exists.

On host:

~~~
touch /mnt/vm-share/TEST-FILE
~~~

On VM:

~~~
ls /mnt/vm-share
~~~

You should see TEST-FILE inside the VM.
