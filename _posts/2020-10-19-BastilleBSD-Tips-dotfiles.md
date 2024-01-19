---
layout: post
title: "FreeBSD 12 - BastilleBSD, dotfiles using xstow"
categories: [BastilleBSD, dotfiles, FreeBSD12]
---

This how I use Git and [Xstow](https://www.freshports.org/sysutils/xstow){:target="_blank"} to manage my dotfiles across [BastilleBSD](https://bastillebsd.org){:target="_blank"}
jails.

[1. Preamble](#preamble)

[2. Host setup](#host-setup)

[3. Bastille template examples](#bastille-template-examples)

[4. Conclusion & thoughts](#conclusion-and-thoughts)

---

# Preamble

I mount my 'dotfiles' for jails in a shared folder (/usr/local/jails/share). This allows me to update my userland
configs across all jails with a single git pull. After looking through my setup I am sure you could incorporate
any other 'dotfile' system you prefer. You could bypass this whole idea and place your jail 'dotfiles' in each of
your Bastille templates. That is a simple solution but it scales poorly once you have a handful of jails.
If you are often tweaking your 'dotfiles' and [TUI](https://github.com/rothgar/awesome-tuis){:target="_blank"}, Git + Xstow allows your latest
changes to be updated across all your jails easily, keeping your TUI consistent across your jails and host.

# Host setup

Make the shared folder where we will store the 'dotfiles'.
~~~
mkdir -p /usr/local/jails/share
~~~

Clone some example dots!
~~~
git clone https://github.com/adriel-tech/stow-example /usr/local/jails/share/dotfiles
~~~

In the future, you can sync these 'dotfiles' with your latest changes.
~~~
git pull /usr/local/jails/share/dotfiles
~~~

# Bastille template examples

This is going to be a template "snippet-dots/Bastillefile"
~~~
# Setup jail home directory
CMD mkdir /root/.config
CMD mkdir /root/.dotfiles
CMD mkdir -p /root/.local/bin
CMD mkdir -p /root/.local/share
# Mount shared dotfiles from host as READ ONLY.
MOUNT /usr/local/jails/share/dotfiles root/.dotfiles nullfs ro 0 0
# Install tui programs
PKG git-lite htop neovim tree xstow
# Setup dotfiles, these were mounted earlier.
CMD xstow -d /root/.dotfiles bin-files
CMD xstow -d /root/.dotfiles git
CMD xstow -d /root/.dotfiles nvim
CMD xstow -d /root/.dotfiles tcsh-root
~~~

<p align="center" width="100%">
    <img src="/assets/images/posts/2020-10-19-BastilleBSD-Tips-dotfiles/wut.gif"> 
</p>

What did we just look at?
- Made some needed folders for roots home directory.
- Mount hosts shared 'dotfiles' into jail /root/.dotfiles as READ ONLY.
- Install programs: "git-lite, htop, neovim, tree, xstow".
- Use xstow to enable our 'dotfiles' 'bin-files, git, nvim, tcsh'

We can include the above snippet at the top of all our Bastillefiles.
~~~
INCLUDE /usr/local/bastille/templates/MY-TEMPLATES/snippet-dots
~~~ 

<p align="center" width="100%">
    <img src="/assets/images/posts/2020-10-19-BastilleBSD-Tips-dotfiles/blocks7.png"> 
</p>

Now if we build a new jail, our shared 'dotfiles' will be mounted
read only in each jails /root/.dotfiles. If you make changes to your 'dotfiles'
on the host or pull in remote changes, the updates will hit all configs across your jails.
If you find a new TUI-TOOL and want to integrate it, edit "snippet-dots/Bastillefile"
and rebuild the jail. Or use bastille cmd to install the app and manually use xstow.

Assuming your new TUI-TOOL configs are already in /usr/local/jails/share/dotfiles
~~~
bastille cmd JAIL-NAME pkg install TUI-TOOL && xstow -d /root/.dotfiles TUI-TOOL
~~~

This can also be used for non root users.

# Conclusion and thoughts

This is the basics of how I use Git + xstow to share 'dotfiles' across jails. In the future
I will share my 'modern' setup which includes these ideas + a modified default jail template.
If this was useful to you, you have a better system or you hate what I have done here...
[let me know.](https://adriel-tech.github.io/contact){:target="_blank"}