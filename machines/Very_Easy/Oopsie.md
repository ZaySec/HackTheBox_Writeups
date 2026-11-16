
>[!tldr] Oopsie is a very easy Linux machine that highlights the impact of information disclosure and broken access control in web applications. Website enumeration reveals a guest login with manipulatable cookies and user IDs allowing escalation to an admin role and access to a file upload feature. A PHP reverse shell is then uploaded to gain an initial foothold. Further enumeration exposes hardcoded credentials enabling lateral movement to another user. Finally, privilege escalation is achieved by abusing a misconfigured SUID binary through PATH hijacking. 


# Guided Mode
---
1. `With what kind of tool can intercept web traffic?`
	1. Flag #1 -> `proxy`

---
2. `What is the path to the directory on the webserver that returns a login page?`
	1. I used `ZAP` instead of burp just because of the automated scan tool
	2. Flag #2 -> `/cdn-cgi/login`

---
3. `What can be modified in Firefox to get access to the upload page?`
	1. After logging in as a guest the `cookies` are shown as "2233"
	2. Flag #3 `cookie`

---
4. `What is the access ID of the admin user?`
	1. Knowing that the guest user has a cookie of "2233" I decided to brute force the cookies to see if we can get any other users. 
		- Testing:
			- `admin` 0-9999
			- `admin@megacorp.com` 0-9999
			- `guest` 0-9999 
	- After going back to the main page the URL I can change the value in `admin.php?content=accounts&id=2` from 2 -> 1
		- `admin.php?content=accounts&id=1`
	- This then got me the admin page
- Flag #4 -> `34322`

---
5. `On uploading a file, what directory does that file appear in on the server?`
	1. In the "Inspector" tool i can modify the 
		1. "role" -> `admin` 
		2. "user" -> `34322`
	2. Then refresh the page or select the uploads tab, either way we are getting admin.
	3. I fuzzed via `Zap` and one of the directories was `/upload` which is the flag
- Flag #6 -> `/uploads`

---
6. `What is the file that contains the password that is shared with the robert user?`

- Now that I know uploads directory and `Content-Type: application/x-php` I can put in a reverse php shell. With my listener up I just need to call the file which *should* give me a shell. 
```shell
------geckoformboundary19414bc4012a7d51fc5ddc5adaad0315
Content-Disposition: form-data; name="name"

revFixed
------geckoformboundary19414bc4012a7d51fc5ddc5adaad0315
Content-Disposition: form-data; name="fileToUpload"; filename="php-reverse-shell.php"
Content-Type: application/x-php

<?php
<SNIP>
```
- Here is part of the GET request to upload the file. I did a some enumeration and here are the findings:
	- User Flag Location: `/home/robert/user.txt` -> `[REDACTED_FLAG]` 
	- Term User: `whoami` -> `www-data`
	- New Cookie: `/var/www/html/cdn-cgi/login/admin.php` -> `86575`
	- Admin Password: `/var/www/html/cdn-cgi/login/index.php` -> `admin`:`MEGACORP_4dm1n!!`
	- SQL Creds: `/var/www/html/cdn-cgi/login/db.php` -> 
		- User: `robert`:`M3g4C0rpUs3r!` 
		- DB:`garage`
- Flag #7 -> `db.php`

---
7. `What executible is run with the option "-group bugtracker" to identify all files owned by the bugtracker group?`
	1. With a simple google search I found the tool `find`.
- Flag #8 -> `find`

---
8. `Regardless of which user starts running the bugtracker executable, what's user privileges will use to run?`
	1. Im going to run this command from `robert`, as we have found similar credentials from earlier. 
		1. Creds: `robert`:`M3g4C0rpUs3r!`
		2. Command: `su robert` -> `M3g4C0rpUs3r!`
	2. Next, to find what privileges it runs we can check with `ls -la <bugtracker>`
```shell
robert@oopsie:/$ ls -la /usr/bin/bugtracker
ls -la /usr/bin/bugtracker
-rwsr-xr-- 1 root bugtracker 8792 Jan 25  2020 /usr/bin/bugtracker
```
- As we can see bugtracekr is ran as root... This can be exploited!
- Flag #8 -> `root`

---
9. `What SUID stands for?`
	- With a quick google search we learn its "Set User ID", however this isn't the answer. So i had to search up specifically what it is
- Flag #9 -> `Set owner User ID`

---
10. `What is the name of the executable being called in an insecure manner?`
	1. When you run `bugtracker` it uses `cat` to read from the reports. However, this question made me realize that not only does `bugtracker` run with `root` priveleges, but also we can use `Path Traversal`

```shell
robert@oopsie:/$ bugtracker
------------------
: EV Bug Tracker :
------------------
Provide Bug ID: 0
0
---------------

cat: /root/reports/0: No such file or directory

robert@oopsie:/$ bugtracker
------------------
: EV Bug Tracker :
------------------
Provide Bug ID: ../root.txt
../root.txt
---------------

[REDACTED_FLAG]
```
- We just got root flag from this!

- Root Flag ->  `[REDACTED_FLAG]`