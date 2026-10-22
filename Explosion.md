>[!tldr] Explosion is a very easy Windows machine that showcases the Remote Desktop Protocol ( #RDP ), how to interact with it using the xfreerdp client and how improper configurations can allow unauthenticated access.

---
1. First i ran `nmap` -> `nmap 10.129.1.13 -sCV`
	- Output -> 
```Shell
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-05-31T21:27:13+00:00; -5s from scanner time.
| ssl-cert: Subject: commonName=Explosion
| Not valid before: 2026-05-30T21:24:18
|_Not valid after:  2026-11-29T21:24:18
| rdp-ntlm-info:
|   Target_Name: EXPLOSION
|   NetBIOS_Domain_Name: EXPLOSION
|   NetBIOS_Computer_Name: EXPLOSION
|   DNS_Domain_Name: Explosion
|   DNS_Computer_Name: Explosion
|   Product_Version: 10.0.17763
|_  System_Time: 2026-05-31T21:27:06+00:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-05-31T21:27:06
|_  start_date: N/A
|_clock-skew: mean: -4s, deviation: 0s, median: -5s
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
```
- Lets learn what the services are
	1. `msrpc` ports:`135,593` : Microsoft Remote Procedure Call  the protocol standard for Windows processes that allows a program running on one host to execute a program on another host.
2. Well since i know this a RDP box ill be trying to connect via port `135`. 
	1. But first i want to try out a script that i found which is related -> `nmap -p 135,593 --script=msrpc-enum <ip>` -> Nothing useful!
	2. `xfreerdp3 /v:<ip> /cert:ignore /dynamic-resolution` -> I quickly realized the major problem of not having a user or password
3. I was informed that i should try low hanging fruit i tried admin, root, user and then `Administrator:` with no password 
4. Flag was just on the desktop
	1. Flag -> `[REDACTED_FLAG]`