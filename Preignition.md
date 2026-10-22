>[!tldr] Preignition is a very easy Linux machine which explores beginner enumeration and directory fuzzing of web applications. Upon finding a vulnerable Admin login endpoint, the web app showcases the dangers of improper configuration and default credentials.

---
1. First I checked if the ip is connected to a website, I then use `ZAP` or the Zed Attack Proxy to capture sent and received data, `ZAP` is also used for enumeration. 
2. I then used `seclists/Discovery/common.txt` to find more directories to explore
	1. Result: `admin.php`
3. I decide to then explore this php page, i quickly try the low hanging fruit of `admin:admin` and it worked
	1. Flag: `[REDACTED_FLAG]`
