# Support Write-up

![Picture of the room](/Medium_rooms/Support/Screenshots/00_Room.png)

We are tasked to pentest the new internal _**Support Operations Platform**_ deployed to assist IT and helpdesk. Security had not the primary focus, but the Platform uses APIs for user Management. Since APIs are likely to be vulnerable, let's see how much we can exploit, and maybe even gain potential Remote Code Execution - RCE.

## Reconnaissance and enumeration

Right of the broad, I decided to run a nmap scan and a gobuster directory enumeration

**Nmap Scan**
```bash
nmap -sV -sC MACHINE_IP
```

_Results_
```txt
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 b8:1b:4e:b5:97:00:e5:4b:bc:07:1a:63:89:ff:74:08 (ECDSA)
|_  256 d2:42:b1:80:cb:6a:80:97:21:62:f3:52:05:69:c2:59 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Support Operations Panel
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

**Gobuster Scan**

```bash
gobuster dir -u http://10.114.180.250 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -r -x.php,.txt
```

_Results_

![Gobuster Results](/Medium_rooms/Support/Screenshots/01_Gobuster.png)

While the Scans where running, I decided to visit the page web Application

![Web page](/Medium_rooms/Support/Screenshots/02_Web.png)

I quickly inspected the Source code to see if there where any credentials left as a comment, but there weren't any. I then checked the pages my Gobuster scan found. Some of them redirected me to `/index.php`, where the login form is. The others just had some assets in them and didn't reveal any sensitive data. 

Taking a look at the login form, I noticed they provided an internal e-mail called **help@support.thm**, which means this email should have an account registered on that page. Knowing that, I tried to some simple SQL injections like `abc' OR1=1;--`, yet not vulnerable to that. I opened up burp, intercepted the request to the page and gave it to a tool called **sqlmap** to investigate deeper on SQL injections. The password parameter wasn't vulnerable, neither was the email parameter

I then tried to brute force the password for the help email. For this, I used a tool called [**ffuf**](https://github.com/ffuf/ffuf), which is a great tool for brute forcing any kind of web related stuff. It's flags are pretty similar to the flags of the `curl` command. I tried following command

```bash
ffuf -w /usr/share/wordlists/rockyou.txt -X 'POST' -d 'email=help@support.thm&password=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -u http://10.114.180.250/index.php -fc 200
```

Luckily, rate limiting wasn't implemented, so this worked really well! To quickly sum up what this command does, it tries every entry in the **rockyou.txt** wordlist as the password for the help e-mail and reports which entry resulted in a different HTTP response other than 200 OK. It works well, since the login likely redirects you once you entered correct credentials, which would be a HTTP status of 302. The tool quickly found a password

![Ffuf results](/Medium_rooms/Support/Screenshots/03_ffuf.png)
> Please note: In real life assessments, brute forcing will take longer, as the rockyou.txt wordlist has 14 million entries. ffuf shares it's progress every second, which will overload your terminal. In such cases, I would recommend writing all of the ffuf results in a file (using shell operator `>>`) and after the tool is done, `grep` that file for Status codes like 302


We'll use these credentials to log in now.

![Ffuf results](/Medium_rooms/Support/Screenshots/04_Dashboard.png)

I noticed there was a feature to change the theme of the Skin. Changing the theme changed the URL and added a query parameter to it. I wanted to see what would happen if I changed the parameter value to `config.php` or `api.php`, files I couldn't access, even after logging in. This resulted in nothing. I remembered Gobuster reported a directory called `/skins`, where all of the themes were located. I tried again with path traversal, instead of `?skin=config.php`, I tried `?skin=../config`, yet nothing again. 

Investigating the Network request, I noticed a new cookie besides the previous PHPSESSID cookie called **isITUser**. I actually wanted to make a new Gobuster scan again on the current path `/dashboard.php`, so I had to look for the cookies for Gobuster to actually scan the page. I tried some more stuff with which I don't want to bore you. After some while, I decided to see if the values of the cookies actually decode to something useful.

I went to [hash_identifier](https://hashes.com/en/tools/hash_identifier) to check whether these cookies are hased or just have a random value

![Identified](/Medium_rooms/Support/Screenshots/05_hashes.png)

Now MD5 is really interesting, as it is an insecure hashing algorithm. I proceeded on decrypting the hash to see what it translates to

![Decrypted hash](/Medium_rooms/Support/Screenshots/06_decrypt.png)

Looks like the hash translates to false, meaning `isITUser` has a value of false. What if we change the value to true? We would just need to encrypt true to it's MD5 hash and check if something changes now

![Access to api.php](/Medium_rooms/Support/Screenshots/07_api-access.png)

We can now access the `/api.php` page, `/config.php` is still not visible for us though. However, this api really looks vulnerable. We could test for 2 vulnerabilities here: BOLA (Broken Object Level Authorization), which is basically IDOR (Insecure Direct Object Reference) just for APIs, or Mass assignment, so changing or own role to admin.

I tried BOLA first, changing the ID of the user to `/user/1` gave us the admin e-mail, which we could potentially brute force again. I also tried mass assignment to set `admin:true` for the help account with a PATCH request, but that failed. 

![E-mail of admin](/Medium_rooms/Support/Screenshots/09_admin_email.png)

I wasted a lot of time trying to brute force the password, so I went back to the `/dashboard` site. Since changing the Cookie changed the Helpdesk site and added the option to view the API, I tried earlier approaches again, and turns out path traversal actually works now! Changing the skin Parameter value to `?skin=../config` actually shows the content of that file! I only found that out of frustration, as I was stuck for quite some while, that I just decided to retry old approaches. 

![Content of config.php](/Medium_rooms/Support/Screenshots/08_admin-pass.png)

Now that we have all the credentials needed, let's log in as the admin. Sadly it didn't work. I thought I copied the credentials wrongly, but I didn't. Some trial and error later, I figured you had to remove the `@` from the password. 

![Logged in as admin](/Medium_rooms/Support/Screenshots/10_admin-access.png)

Finally, that only took a lot of my nerves, let's hope the last flag will be easier. And easier it was indeed. I noticed there is a new feature now next to the **Select Theme** button. I changed `Date` to `time`, and it showed the current UTC time. The URL didn't change at all, so I checked my developer tools to see if a request was made when changing it to time. 

I noticed a POST request, looked at the request body and noticed this

![POST request](/Medium_rooms/Support/Screenshots/11_time.png)

A parameter called `sys` executing the `date` command. If isn't vulnerable to Command injection, I would've lost my mind. I opened Burp, intercepted that request and changed the parameter value to `whoami`. At first, it didn't work

![First attempt](/Medium_rooms/Support/Screenshots/12_try1.png)

However, that doesn't mean it's not vulnerable to command injection. I tried to trick the application to think it would execute the date command, but added the `whoami` command to it as well using the `;` shell operator

![First attempt](/Medium_rooms/Support/Screenshots/13_try2.png)

Bingo, the application executed the command now, so that means, all we have to do to get our flag now is:

![First attempt](/Medium_rooms/Support/Screenshots/14_final-flag.png)

We could now establish a reverse shell and try to escalate our privileges to gain full access over the system. However, this isn't part of our assessment! 

## Conclusion

This room pushed me and my nerves to my limit. I was stuck on here for quite some while, especially after logging in as the help account for the first time. I already suspected the URL query parameter to be vulnerable to Local file inclusion, so I was really devastated when it didn't work at first. However, after I figured that out, the rest was pretty easy! 

To sum up, we chained following vulnerabilities to gain Remote code Execution:
Brute force the authentication, Cookie Manipulation, API BOLA, Local File Inclusion, and finally command injection

Thank you so much for reading til the end, I hope this write-up helped you out! 
