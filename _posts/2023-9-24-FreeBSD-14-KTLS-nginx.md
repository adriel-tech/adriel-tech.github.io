---
layout: post
title: "FreeBSD 14 - enable Kernel TLS for nginx"
categories: [FreeBSD14, nginx]
---

This is how to enable Kernel TLS on FreeBSD 14 for use with nginx.
This article assumes you already use nginx and you generally know how
to use FreeBSD.

FreeBSD 14 loads KTLS by default unlike FreeBSD 13.

## Enable ktls sysctl

~~~
# sysctl kern.ipc.tls.enable=1
~~~

## Add required sysctl.conf settings to enable at boot.

~~~
echo "kern.ipc.tls.enable=1" >> /etc/sysctl.conf
~~~

## Edit nginx configuration

make sure sendfile is on.

/usr/local/etc/nginx.conf
~~~
http {
    sendfile on;
}
~~~

add the following lines into your nginx configurations server blocks.

/usr/local/etc/nginx.conf
~~~
server {
    ssl_conf_command Options KTLS;
    ssl_protocols TLSv1.3;
}
~~~

## Restart nginx

~~~
# service nginx restart
~~~

## View ktls stats

~~~
# sysctl kern.ipc.tls.stats
~~~

