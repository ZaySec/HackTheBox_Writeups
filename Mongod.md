>[!tldr] Mongod is a very easy Linux machine which introduces the #MongoDB database and how to use the MongoDB shell utility to interact with it. Misconfiguring the database by leaving the anonymous user enabled, allows exploitation and full enumeration of the database.

# Adventure Mode
---
1. To start i run a simple nmap command `nmap <ip> -sCV`
```Shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
2. I already know that it should be related to the mongoDB so i expand the ports the nmap check (*Nmap scans the most common 1,000 ports for each protocol*). I ended up appending `-p-` which should check all ports -> 
	1. `nmap <ip> -sCV -p-`
	2. `nmap <ip> -sCV -p27017,27018,27019,27020`: Common MongoDB ports
	- They both outputted a lot but i know that 27018,27019,27020 arent open so i wont use them, and `27017` has more info (and a LOT more info)
```shell
27017/tcp open   mongodb MongoDB 3.6.8 3.6.8
| mongodb-databases:
|   databases
|     0
|       empty = false
|       sizeOnDisk = 32768.0
|       name = admin
|     1
|       empty = false
|       sizeOnDisk = 73728.0
|       name = config
|     4
|       empty = false
|       sizeOnDisk = 32768.0
|       name = users
|     2
|       empty = false
|       sizeOnDisk = 73728.0
|       name = local
|     3
|       empty = false
|       sizeOnDisk = 32768.0
|       name = sensitive_information
|   ok = 1.0
|_  totalSize = 245760.0
| mongodb-info:
|   MongoDB Build info
|     version = 3.6.8
|     openssl
|       compiled = OpenSSL 1.1.1f  31 Mar 2020
|       running = OpenSSL 1.1.1f  31 Mar 2020

<...snip...>
fstack-protector-strong -fno-builtin-memcmp
<...snip...>
|sysInfo = deprecated
|     gitVersion = 8e540c0b6db93ce994cc548f000900bdc740f80a
```
- There is a lot of info from the nmap scan, some important others not so much! 
	- The mongodb version is `3.6.8` -> **[CVE-2025-14847](https://thehackernews.com/2025/12/new-mongodb-flaw-lets-unauthenticated.html)** However, since the machine was created in 2022 i don't think this is the path i should be going.
	- I honestly had a lot of trouble even connecting (im using arch btw) so i had to ask claude -> `sudo docker run -it --rm mongo:3.6 mongo --host 10.129.228.30 --port 27017` This seemed to have gotten me in, further enumeration is required
3. I was having trouble using the commands until i realized that something like `pwd` wouldn't work and i had to use the actual function `pwd()`. 
- Heres some outputs ->
```Mongo
> show dbs
admin                  0.000GB
config                 0.000GB
local                  0.000GB
sensitive_information  0.000GB
users                  0.000GB
```
- `admin` and `sensitive_information` seem pretty useful, ill check out both but first `sensitive_information` since there probably is a flag within.
```Mongo
> use sensitive_information
> show collections
flag
```
4. Ok awesome after some quick enumeration i found the flag, time to figure out how to read it T_T... I had to ask claude again how to read the flag
	1. Flag -> `db.flag.find()` -> `[REDACTED_FLAG]`

---
---
# Guided Mode
---
1. `How many TCP ports are open on the machine?`
	1. With a simple `nmap` scan i can get the ports
		1. `sudo nmap -sS -p- 10.129.50.95 -Pn`
- Flag #1 -> `2`

---
2. `Which service is running on port 27017 of the remote host?`
	1. To get the service i used `nmap` again
		1. `sudo nmap -sCV -p27017 10.129.50.95 -Pn > nmap-output.txt`
		2. `cat nmap-output.txt | less` -> To read output
- Flag #2 -> `MongoDB 3.6.8`

---
3. `What type of database is MongoDB? (Choose: SQL or NoSQL)`
	1. With a quick google search i find its not sql
- Flag #3 -> `NoSQL`

---
4. `What command is used to launch the interactive MongoDB shell from the terminal?`
	1. The tool to use is `mongosh`
		1. `mongosh "mongodb://10.129.50.95:27017"`


## Stupid ass tool dont work on arch