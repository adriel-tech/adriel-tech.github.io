---
layout: post
title: "FreeBSD - Things I do on a fresh install"
categories: [FreeBSD]
---

* Updated January 2025

This is a general list of things I do and software I install on a fresh install of FreeBSD. 
I will update this periodicly, currently this applies to FreeBSD 13 using ZFS. All commands
assume you are root, use sudo/doas as needed.

🚨 Use this as a reference, don't blindly use it!

## PKG

Change to the "latest" PKG repository.

~~~
mkdir -p /usr/local/etc/pkg/repos/
~~~

~~~
cat > /usr/local/etc/pkg/repos/FreeBSD-latest.conf <<EOF
#
# Override /etc/pkg/FreeBSD.conf to latest.
#

FreeBSD: { enabled: no }

FreeBSD-latest: {
  url: "pkg+https://pkg.FreeBSD.org/\${ABI}/latest",
  mirror_type: "srv",
  signature_type: "fingerprints",
  fingerprints: "/usr/share/keys/pkg",
  enabled: yes
}
EOF
~~~

~~~
pkg upgrade -y
~~~

Install the user software I like to use.
~~~
pkg install -y age bash bash-completion broot chezmoi curl fzf git htop iperf3 neovim nmap tmux tree
~~~

Install FreeBSD utilities.
~~~
pkg install -y bastille powerdxx sudo vm-bhyve wireguard-tools zerotier
~~~

## /etc/rc.conf settings

Set some general stuff.
~~~
sysrc clear_tmp_enable="YES"
sysrc syslogd_flags="-ss"
~~~

Disable sendmail.
~~~
sysrc sendmail_enable="NO"
sysrc sendmail_submit_enable="NO"
sysrc sendmail_outbound_enable="NO"
sysrc sendmail_msp_queue_enable="NO"
~~~

Enable ntpd.
~~~
sysrc ntpd_enable="YES"
sysrc ntpd_sync_on_start="YES"
~~~

Enable powerd++ and set cpu c-states.
~~~
sysrc powerd_enable="NO"
sysrc powerdxx_enable="YES"
sysrc powerdxx_flags="-a hiadaptive"
sysrc performance_cx_lowest="Cmax"
sysrc economy_cx_lowest="Cmax"
~~~

Enable PF firewall.
~~~
sysrc pf_enable="YES"
sysrc pf_flags=""
sysrc pf_rules="/etc/pf.conf"
sysrc pflog_enable="NO"
sysrc pflog_flags=""
sysrc pflog_logfile="/var/log/pflog"
touch /var/log/pflog
~~~

Enable ssh.
~~~
sysrc sshd_enable="YES"
sysrc sshd_dsa_enable="NO"
sysrc sshd_ecdsa_enable="NO"
~~~

## Filesystem related

### Merge /tmp and /var/tmp

~~~
mv /var/tmp/* /tmp/
zfs destroy -r zroot/var/tmp
ln -s /tmp /var/tmp
~~~

### Move some logs to tmpfs in RAM

On systems with enough ram and using an SSD, this also requires modification to /etc/newsyslog.conf
~~~
mkdir /var/log/RAM
echo 'tmpfs           /var/log/RAM tmpfs rw,size=250m,late 0 0' >> /etc/fstab
~~~

/etc/newsyslog.conf
~~~
/var/log/RAM/all.log                 600   0      5000  *      C
/var/log/RAM/amd.log                 644   0      100   *
/var/log/RAM/console.log             600   0      5000  *      C
/var/log/RAM/cron                    600   0      5000  *      C
/var/log/RAM/daemon.log              640   0      5000  *      C
/var/log/RAM/debug.log               600   0      5000  *      C
/var/log/RAM/devd.log                644   0      5000  *      C
/var/log/RAM/init.log                644   0      5000  *      C
/var/log/RAM/kerberos.log            600   0      5000  *      C
/var/log/RAM/lpd-errs                644   0      5000  *      C
/var/log/RAM/maillog                 640   0      5000  *      C
/var/log/RAM/ppp.log  root:network   640   0      5000  *      C
/var/log/RAM/sendmail.st             640   0      5000  *      CBN
/var/log/RAM/xferlog                 600   0      5000  *      C
~~~

Delete logs and restart newsyslog
~~~
rm -rfv /var/log/*
service newsyslog restart
~~~