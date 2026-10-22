>[!tldr] Appointment is a very easy Linux machine which showcases beginner #SQL Injection techniques against an SQL database enabled web application.

---
1. Started off with a nmap scans 
	1. `nmap 10.129.3.213 -sT -p- --stats-every=10s` -> port `80`
	2. `nmap 10.129.3.213 -sCV -p80` ->
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Login
|_http-server-header: Apache/2.4.38 (Debian)
```
- From the nmap scan we can see this is a webpage with a login screen, so im going to open up `ZAP` and use the automated scan to see what info i can get from there.
- The automated scan got us a lot of info, from that the important pieces are `SQLi` and a vulnerable js library. 
2. With this info i then wanted to use a `SQLi` attack, but since i wasnt sure where to start. I decided to capture the login request, and brute force with a list of `SQLi` techniques. This way i can not only have a clear list of attacks, but also speed up the process
3. To get the request i decided to use a simple injection of `admin' or '1'='1` which i learned from a previous HTB module and that worked so no need for a fuzzing (i did it anyways and it actually wasn't that useful)
	- Flag -> `[REDACTED_FLAG]`

---
