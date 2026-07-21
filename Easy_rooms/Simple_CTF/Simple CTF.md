# Walkthrough Simple CTF
![Picture of the room](/Easy_rooms/Simple_CTF/Screenshots/00_Room.png)

The simple CTF is a beginner level CTF. Judging by the questions, this is a Boot2Root challenge. They require you to scan the host, find a Vulnerability, exploit it and finally gain root access by spawning a shell. This is my first write-up for a CTF, so let's get started!

## Reconnaissance
First, as always, we will need to do some Reconnaissance on the target machine to figure out which ports are open and what services are running on it. We do that with a simple Nmap scan 
```bash
nmap -sS -sV <TARGET_IP>
```

This will give us the following information:

```text
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
```

As we can see, there are 3 ports open, with which we can answer the first two questions. We should keep in mind that SSH is running on port 2222 instead of its usual port, 22. There's an Apache web server running on port 80, Let's access it through a browser.
![picture of the website](/Easy_rooms/Simple_CTF/Screenshots/01_Website.png)

Indeed, there is a web page, the default apache Website. Before looking for a CVE to answer question 3, we should first try to find out more information about the website. So let's run a quick gobuster brute force enumeration, to see if there are any hidden directories or hidden files. If you are not running the attackbox or Kali Linux, you should try to download the dirbuster wordlists

```bash
gobuster dir -u http://10.114.138.142/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x.txt,.php
```
![Picture of the gobuster dir scan](/Easy_rooms/Simple_CTF/Screenshots/02_GobusterScan.png)

We notice two important things from that scan. Firstly, the "Robots.txt" file is avaiable to everyone. If we try to look at the content of that file, we dont see anything interesting there. However, visiting the other found path /simple, we get redirected to another page running "CMS made simple" . Maybe there is a CVE for this service, let's see which version of CMS is running.

![CMS Version info](/Easy_rooms/Simple_CTF/Screenshots/03_CMS.png)

## Vulnerability

So, now we know it's CMS Made Simple version 2.2.8, let's try to look up a known CVE for that version! 

![CVE for CMS version 2.2.8](/Easy_rooms/Simple_CTF/Screenshots/04_CVE.png)

Found it! This will be the answer to question 3! After some quick research on the CVE, we can see it's vulnerable to a SQL Injection, which answers question 4 as well. Now that we know what it's vulnerable to, we can search for an exploit. I found one on GitHub and downloaded it. [CVE-2019-9053 exploit](https://github.com/Perseus99999/CVE-2019-9053-working-/blob/main/exploit.py). I then made the exploit executable and ran it with the needed flags. (Again, if you dont run the attackbox or Kali Linux, download the rockyou.txt wordlist if you haven't already!)


## Exploitation

```bash
chmod +x exploit.py
exploit.py -u http://TARGET_IP/simple/ -c -w /usr/share/wordlists/rockyou.txt 
```
!Please note that the exploit might take some time until it finds the password!
After the exploit is done, we get a lot of valuable information

![Result of the Exploit](/Easy_rooms/Simple_CTF/Screenshots/05_ExploitResult.png)

With that, we can answer question 5! I firstly tried to find another hidden directory using another gobuster dir mode enumeration, and I found one, this being /admin. I was also able to login with the obtained details, but there wasn't anything special and also didn't answer question 6. After some confusion, I remembered that ssh is running on port 2222, which I even stated to keep in mind at the beginning of this write-up, silly me! So, I opened up terminal again and ssh'd into the machine using the credentials

```bash
ssh -p 2222 REDACTED@10.114.138.142
```

### And we are in!

## Post Exploitation
Now that we have access to the host through user REDACTED, we need to escalate our privileges! First of all, for the user flag, we can list the content from REDACTED's home and submit the contents of the file as the answer to question 7

![first flag](/Easy_rooms/Simple_CTF/Screenshots/06_user_flag.png)

Okay, now for the next question, question 8, we need to figure out if there is another user, we can run

```bash
ls -la /home
```
There will be another user listed, that will be the answer to question 8. Now, for the last 2 questions, 9 and 10, we need to run commands as root in order to view the content of root user's file. First of all, let's check whether our current user can execute commands as root. To check that, type in:
```bash
sudo -l
```
We can see we can run the Binary vim in Unix System Resources "/usr/bin/vim" as root, so we can use that to answer question 9!
Sadly, I don't have any prior experience with vim and had no idea how to use the program. I tried running 'sudo vim' first to see what it is about. I got stuck there and couldn't exit it, so I had to connect to the machine via ssh again. I looked up what the purpose of vim is and found out it is a keyboard driven program used to write code. I searched "How to spawn a privileged shell using vim" and found out, you could run 'vim -c' and adding the shell type. Because I tried earlier to view the content of /root with ls, I got an error message from the shell which reported "permission denied", which included the Vim shell escape '!sh'. So the final command was simply:

```bash
sudo vim -c '!sh' 
```
After that, I was able to run commands as root and check the root user's content

![root flag](/Easy_rooms/Simple_CTF/Screenshots/07_user_flag.png)

And there it is. the root flag, the answer to the final question!

## Conclusion
Well, that was a fun and easy CTF! The third one I solved so far. I am happy to see I am getting better. I even learned something new from that CTF! I am looking forward to writing more write ups in the future. I tried to make this write-up as detailed as possible, so beginners can follow along without being confused. Hopefully this write-up helped you out! :D
