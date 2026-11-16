>[!tldr] Fawn is a very easy Linux machine which explores the File Transfer Protocol ( #FTP ) and its exploitation when misconfigured to allow anonymous access.

---
1. Started off with a nmap scan `nmap <target> -A`
>[!info] Nmap Output
>```
>PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
>```

- As we can see service `FTP` is open with *Anonymous FTP login* allowed. This means we can use the credentials `anonymous:` to login. We can also see `flag.txt` is exposed.
2. I used `ftp <target> -p 21` to connect. Then used the anonymous credentials which worked
3.  Now successfully logged in, a quick `ls` command displayed `flag.txt` again. So i used `get flag.txt` to download the file from the FTP server.
4. Next in my local Linux terminal I concatenated the file to get the flag with `cat flag.txt`. Which worked!