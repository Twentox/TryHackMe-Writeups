- after downloading the ZIP file and extracting its contents, we can see three different files:
```bash
drwxr-xr-x 2 erik erik   4096 Nov  3 17:59 .
drwxr-xr-x 3 erik erik   4096 Nov  3 17:59 ..
-r--r--r-- 1 erik erik 108653 Apr 11  2021 access.log
-r--r--r-- 1 erik erik  22671 Apr 11  2021 auth.log
-r--r--r-- 1 erik erik   3213 Apr 11  2021 vsftpd.log
```


## Reconnaissance: 
---

#### What tools did the attacker use?: 
---
 - the first log file that comes to mind is `access.log`, because this file logs all HTTP requests sent to the web server 
 - to find out which tools were used, it is often worth looking at the `User-Agent` header in the requests 
 - for example in this request: 
 ```bash
 ::ffff:192.168.10.5 - - [11/Apr/2021:09:23:32 +0000] "GET /rest/user/whoami HTTP/1.1" 200 11 "http://192.168.10.4/" "Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0"
 ```

- we can see that the `User_Agent` is `Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0` 
- because the `User-Agent` header seems to be the last field in the line, we can use `awk` and `uniq` to extract the unique User-Agent values 
```bash
cat access.log | grep -v "Mozilla" | uniq
```

```bash
"-"
<redacted>
Firefox/78.0"
<redacted>
Firefox/78.0"
<redacted>
Firefox/78.0"
<redacted>
<redacted>
Firefox/78.0"
```

#### What endpoint was vulnerable to a brute-force attack?: 
----
- when I think of brute-force attacks against web servers, I usually think of POST requests targeting a login page
- so let's use `grep` again to filter for these two keywords:
```bash
cat access.log | grep "POST" | grep "login"
```

```bash
...
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted> HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted> HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
::ffff:192.168.10.5 - - [11/Apr/2021:09:16:30 +0000] "POST <redacted>  HTTP/1.0" 401 26 "-" "Mozilla/5.0 (Hydra)"
...
```

- based on the `User-Agent` header, we can see that Hydra was used to send these POST requests
- this confirms that the attacker tried to `brute-force` the endpoint 

#### What endpoint was vulnerable to SQL injection?: 
---
- as we saw in the first task, `sqlmap` was used in these requests
- this tool is used to automate SQL injection attacks
- to answer the question, we have to use `grep` to filter for requests containing `sqlmap`: 
```bash
cat access.log | grep sqlmap 
```

```bash
...
::ffff:192.168.10.5 - - [11/Apr/2021:09:29:16 +0000] "GET <redacted>?q=1%20AND%203813%20IN%20%28SELECT%... HTTP/1.1" 200 30 "-" "sqlmap/1.5.2#stable (http://sqlmap.org)"
...
```

- we can see that the `q` parameter is used to send URL-encoded data
- if we decode the data in the first request in the `code-block` above we can see this: 
```bash
1 AND 3813 IN (SELECT (CHAR(113)+CHAR(122)+CHAR(107)+CHAR(113)+CHAR(113)+(SELECT (CASE WHEN (3813=3813) THEN CHAR(49) ELSE CHAR(48) END))+CHAR(113)+CHAR(106)+CHAR(120)+CHAR(112)+CHAR(113)))
```

- that confirms our finding 

#### What parameter was used for the SQL injection?: 
---
- this was mentioned in the last task 

#### What endpoint did the attacker try to use to retrieve files?: 
----
- Since we also have a `vsftpd.log`, I initially assumed that we were looking for an endpoint related to FTP 
- lets filter for it with `grep`: 
```bash
cat access.log | grep ftp
```

```bash
::ffff:192.168.10.5 - - [11/Apr/2021:09:34:33 +0000] "GET /<redacted> HTTP/1.1" 200 4852 "-" "feroxbuster/2.2.1"
::ffff:192.168.10.5 - - [11/Apr/2021:09:34:40 +0000] "GET /<redacted>/www-data.bak HTTP/1.1" 403 300 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0"
::ffff:192.168.10.5 - - [11/Apr/2021:09:34:43 +0000] "GET /<redacted>/coupons_2013.md.bak HTTP/1.1" 403 78965 "-" ""Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0"
```

## Stolen data: 
---

#### What section of the website did the attacker use to scrape user email addresses?: 
---
- to find the answer, I used `grep`, `awk` and `uniq` to extract the different endpoints: 
```bash
cat access.log | awk '{print $7}' | uniq | grep -v "search"
```

- after reviewing the output, I found several interesting endpoints: 
```bash
/rest/user/whoami
/rest/<redacted>/9/<redacted>
/rest/user/whoami
/rest/<redacted>/20/<redacted>
/rest/admin/application-configuration
/rest/admin/application-version
/rest/admin/application-configuration
```

#### Was their brute-force attack successful? If so, what is the timestamp of the successful login?: 
---
- to check whether the brute-force attack was successful, we have to search for POST requests with a User-Agent header containing `Hydra` and an HTTP status code of `200` or `302`
- a status code of `302` is worth checking because login pages often redirect users after a successful login 
- lets use `grep` again: 
```bash
cat access.log | grep "POST" | grep "Hydra" | grep "200"
```

```bash
::ffff:192.168.10.5 - - [<redacted>] "POST /rest/user/login HTTP/1.0" 200 831 "-" "Mozilla/5.0 (Hydra)"
```

- I also tried searching for status code `302`, but there were no matching results
- Therefore, I used `200` instead

#### What user information was the attacker able to retrieve from the endpoint vulnerable to SQL injection?: 
---
- after going through some of the `requests` related to the `SQLi` attack, I found a request that looked promising: 
```bash
::ffff:192.168.10.5 - - [11/Apr/2021:09:32:51 +0000] "GET /rest/products/search?q=qwert%27))%20UNION%20SELECT%20id,%20email,%20password,%20%274%27,%20%275%27,%20%276%27,%20%277%27,%20%278%27,%20%279%27%20FROM%20Users-- HTTP/1.1" 200 3742 "-" "curl/7.74.0"
```

- If we decode the payload, we can identify the database columns being targeted: 
```bash
qwert')) UNION SELECT id, <redacted>, <redacted>, '4', '5', '6', '7', '8', '9' FROM Users--
```

#### What files did they try to download from the vulnerable endpoint?: 
---
- the hint says: `What other log files do you have?` 
- so im guessing we should search in the `vsftpd.log`

- after searching for 10 seconds I found the filenames at the very bottom of the `vsftpd.log`: 
```bash
Sun Apr 11 09:35:45 2021 [pid 8154] [ftp] OK DOWNLOAD: Client "::ffff:192.168.10.5", "<redacted>", 2602 bytes, 544.81Kbyte/sec
Sun Apr 11 09:36:08 2021 [pid 8154] [ftp] OK DOWNLOAD: Client "::ffff:192.168.10.5", "<redacted>", 131 bytes, 3.01Kbyte/sec
```

#### What service and account name were used to retrieve files from the previous question?: 
---
- if we look through the `vsftpd.log`, wen can see plenty of `OK LOGIN`-Messages 
- here we can find the `username` and the service should be obvious 

#### What service and username were used to gain shell access to the server?: 
---
- here we have to use the `auth.log`, to find the service and username 
- if we look through the file we see a lot of `SSH connection`-Messages 
- to get the username that logged in through `SSH`, we have to search for `SSH`-messages that have `Accepted` in them: 
```bash
cat auth.log | grep ssh | grep Accepted
```

```bash
Apr 11 09:41:19 thunt sshd[8260]: Accepted password for <redacted> from 192.168.10.5 port 40112 ssh2
Apr 11 09:41:32 thunt sshd[8494]: Accepted password for <redacted> from 192.168.10.5 port 40114 ssh2
```