## Nmap: 
---
```bash
nmap -T4 -n -A 10.67.147.204 -oN nmap
```

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-23 17:53 CET
Nmap scan report for 10.67.147.204
Host is up (0.11s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 24:80:a6:35:46:63:e0:dd:cc:c8:f1:74:d9:83:c9:bc (RSA)
|   256 bf:1f:73:25:2a:d8:e9:e3:e5:03:b8:7c:d5:68:5d:52 (ECDSA)
|_  256 0a:a1:4f:b2:47:4f:76:e1:af:4e:55:b5:44:45:c9:b0 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.26 seconds
```
## Gobuster: 
---
- when i opened the web-page i saw the `Apache` default-page
- I didn't see anything useful in the source-code of that default-page
- so lets use `gobuster` so see if there are any files or directory's:
```bash 
gobuster dir -w /usr/share/SecLists/Discovery/Web-Content/common.txt -u http://10.67.147.204 -o gobuster/first_enum
```

```bash 
/.hta                 (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/app                  (Status: 301) [Size: 312] [--> http://10.67.147.204/app/]
/index.html           (Status: 200) [Size: 10918]
/server-status        (Status: 403) [Size: 278]
```
- lets go to `/app` so see whats there 