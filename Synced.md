>[!tldr] Synced is a very easy Linux machine that explores the #Rsync protocol, how to interact with it and the dangers of improper configuration allowing anonymous authentication.

# Adventure Mode
---
1. Starting off with a nmap scan `nmap <ip> -sCV` ->
	- I also tried `rsync-list-modules` script didn't give more info
```shell
PORT    STATE SERVICE VERSION
873/tcp open  rsync   (protocol version 31)
```
- After some research `rsync` is "a utility for transferring and synchronizing files between a computer and a storage drive and across networked computers". Basically a protocol to send files. 
	
2. I also tried enumerating with `msfconsole` but it gave me less than useful output ->
```shell
msf auxiliary(scanner/rsync/modules_list) > run
[-] 10.129.228.37:873     - does not appear to be rsync
[*] 10.129.228.37:873     - Scanned 1 of 1 hosts (100% complete)
```
- I think ill have to ignore this info and continue with another enumeration technique
	
3. Luckily just using `rysnc` i found out about its public anonymous share
```shell
❯ rsync 10.129.228.37::
public         	Anonymous Share
```
- From here I need to learn how to login is anonymous or get more info from this
	
4. I found the following command and output, we just need to read from the share
```shell
❯ rsync -av --list-only rsync://10.129.228.37/public

receiving incremental file list
drwxr-xr-x          4,096 2022/10/24 18:02:23 .
-rw-r--r--             33 2022/10/24 17:32:03 flag.txt
```
5. I then try `rsync -avz rsync://10.129.228.37/public Downloads/` and the flag gets downloaded to my `Downloads/` directory. From here i cat `flag.txt` and get
	1. Flag -> `[REDACTED_FLAG]`
