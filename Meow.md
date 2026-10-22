>[!tldr] Meow is a very easy Linux machine which guides players on setting up their attacking machines, connecting to HTB labs via VPN and demonstrates the strategy of how to complete them. The machine focuses on beginner enumeration techniques and showcases the exploitation of a vulnerable Telnet service through default credentials.

---
1. I started off checking the ports with `nmap`
	- `nmap 10.129.1.17 -A`
>[!info] Output
>```
>PORT   STATE SERVICE VERSION 
>23/tcp open  telnet  Linux telnetd
>Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
2. Useful we now know about `port 23` and the `telnet` service. Telnet is a insecure remote connection, since replaced with ssh since its more secure.
3. Commands:
	1. `telnet 10.129.1.17 23` : This connects our terminal to telnet
	- Once i connected in i tried `root:root` which worked and i got the flag.
