---
layout: post
title: "Easy duel booting operating systems"
categories: [FreeBSD, Linux, Windows]
---

Duel booting is a common way to use or test different operating systems with your hardware. Depending on your operating systems
and hardware, it can be a real pain. In this post I will explain some basic rules to make it easier to duel boot
and some useful commands.

## My duel boot rules

1️⃣ Never mix operating systems on the same hard drive. Keeping each OS on it's own hard drive with its own boot loader
is less complicated and removes the issues with operating systems fighting over the boot loader. I have followed this
rule since the days of Windows XP and I stick to it today.

2️⃣ Use your UEFI to boot to different operating systems. You do not need a single boot loader modded to work with different
operating systems. You do not want a bootloader update to lock you out of one of your operating systems, requiring a recovery
disk to modify the boot loader. Instead use your UEFI boot menu and choose what OS you want to boot into, this requires
no setup, no extra configurations and no risk. 

You can turn your PC on and use the proper key combination to enter your UEFI or boot manager. Usually the key is ESC, F12 or F10, 
it can also be DEL or F8, you will have to look through your own computers manual to find out. This allows you to boot into your
non default OS.

Below are commands I use to reboot straight to UEFI from my current operating system so that I may choose a different operating
systems.

FreeBSD
```
efibootmgr -f && reboot
```

Sh alias
```
alias reboot-uefi='efibootmgr -f && reboot'
```

Linux (systemd)
```
systemctl reboot --firmware-setup
```

Bash alias
```
alias reboot-uefi='systemctl reboot --firmware-setup'
```

Windows 10+
```
shutdown /r /fw /t 0
```

Powershell alias
```
function reboot-uefi { shutdown /r /fw /t 0 }
```

You can also search 'windows advanced startup' in the Windows start menu. It will take you to a UI which allows
booting into your UEFI firmware or directly to another OS.

3️⃣ Windows clock. I believe most people duel booting are using windows. Unlike FreeBSD and Linux, Windows
uses an 'RTC' clock and not 'UTC'. If you duel boot Linux with its default settings and reboot back into windows
your windows clock will be out to lunch and cause you some issues.

You can run the below command on your Linux (systemd) boot and set the clock to RTC like windows stopping/preventing
this issue.

```
timedatectl set-local-rtc 1 --adjust-system-clock
```

On FreeBSD during install it will ask you if your machine uses UTC time or not, if duel booting Windows choose not to use UTC.

I hope this is helpful, have fun and try not to nuke the wrong drive when installing your next duel boot test!
