---
title: HackTheBox - Blocky
categories:
- writeups
- todo
layout: post
date: 2020-07-14 00:05 -0500
---
![Blocky](/assets/images/HTBoxes/Blocky/Blocky.png)

## Summary

Blocky is a box that is meant to teach bad password practices, decompiling JAR files, and the perils of exposing internal files on public websites.  I first enumerate with nmap to find SSH and HTTP open as well as some other ports that ended up not being paths to follow.  I then enumerated the website's directories in order to find some java code that exposed credentials.  Those credentials were reused and were used to gain a foothold.  Due to excessive privileges assigned to the normal user, I was able to easily escalate to root immediately after gaining a foothold.

## Walkthrough

### Enumeration

As usual, I started with nmap to scan the machine.  ``nmap -A -T4 -v -p1-30000 10.10.10.37``

```bash
PORT      STATE  SERVICE   VERSION  
21/tcp    open   ftp       ProFTPD 1.3.5a  
22/tcp    open   ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)  
80/tcp    open   http      Apache httpd 2.4.18 ((Ubuntu))  
8192/tcp  closed sophos  
25565/tcp open   minecraft Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)
```

FTP doesn't have anonymous access allowed, so I decided to checkout the webserver.

![Blockycraft](/assets/images/HTBoxes/Blocky/blockycraft.png)

I ran ``gobuster`` against the site, looking for subdirectories.

```bash
gobuster dir -u http://10.10.10.37 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.37
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/07/13 22:11:18 Starting gobuster
===============================================================
/wiki (Status: 301)
/wp-content (Status: 301)
/plugins (Status: 301)
/wp-includes (Status: 301)
/javascript (Status: 301)
/wp-admin (Status: 301)
/phpmyadmin (Status: 301)
```

At about 40% I decided to stop the scan and look at what I found so far.  In particular ``/plugins`` and ``/phpmyadmin`` caught my eye. Attempting defaults root/password or root/null on PHPMyAdmin did nothing so I looked at plugins instead.

The ``/plugins`` page had a couple jar files listed that I could download.

I ran ``strings`` against the jar files quick and found one of them seemed to have references to "my first plugin" so I figured this is the interesting one.

I unzipped the jar file and went to open the ``com/myfirstplugin/BlockyCore.class`` file.  It was clearly not plaintext so I ran ``strings BlockyCore.class`` and noticed a root user followed by its password on the next line:

```bash
--snipped--
<init>
Code
        localhost
root
8YsqfCTnvxAUeduzjNSXe22
LineNumberTable
LocalVariableTable
--snipped--
```

I went back to the PHPMyAdmin console and saw the username / password combo worked.  However, I like to double check my path with the official walkthroughs of HTB because time is limited for me. I noticed it alludes to using JD-GUI to decompile the plugin. So I decided to try this out and get the experience since I haven't done it before.

On my Kali box I looked for JD-GUI but didn't see it.  I did see [JaDx-GUI](https://github.com/skylot/jadx) available so I opened that and opened the jar file.  Sure enough the credentials are hard-coded.

![blockyjar](/assets/images/HTBoxes/Blocky/blockyjar.png)

After looking at the PHPMyAdmin console for a bit I saw there was a wordpress user named ``notch``.

![wordpress](/assets/images/HTBoxes/Blocky/wpusers.png)

### Foothold

I couldn't find any clear path to Foothold with an exploit etc. so I decided to take another peak at the official walkthrough.  It looked like there was a complex path (marked **todo** for later) that involved PHPMyAdmin and an easier path.  Cue the facepalm, as I forgot to try the credentials on the other services I found earlier with ``nmap``.

I logged into the box via SSH with the username ``notch`` I found above in Wordpress users, using the same password found from the jar file.

### User

I didn't need to pivot to any other user, so I just grabbed the user.txt string from ``/home/notch`` and started enumerating.  I ``scp``'d over an enumeration script and ran it (LinEnum) but honestly, it wasn't needed.  I ran ``sudo -l`` and noticed I had access to ALL commands.

```bash
notch@Blocky:~$ sudo -l
[sudo] password for notch:
Matching Defaults entries for notch on Blocky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
```

### Root

Given that notch is essentially an administrator, I just ran ``sudo -i`` and got a root shell, then went and grabbed the root.txt.

### Lessons Learned

The main thing I learned on this box was a different way of accessing jar files using tools that decompile them.  In this case, _JaDx-GUI_.  Overall a pretty simple yet entertaining box and I'm logging off for the evening feeling satisified.

I realized I keep capturing my mouse cursor in screenshots, so I'll try to avoid that next time.
