# Recruit Write-up

![Picture of the room](/Medium_rooms/Recruit/Screenshots/00_Room.png)

**_Recruit_** launched a recruitment portal, yet Management suspects security may have been overlooked during deployment. We are tasked to assess the application, map it's structure and try to exploit vulnerabilities. 

Before we start, please add `recruit.thm` to your `/etc/hosts` file, since the application refers to itself as that domain instead of just using the IP to visit it.

```bash
sudo echo 'MACHINE_IP recruit.thm' >> /etc/host
```

Now, you can access the page through the following URL: `http://recruit.thm`
## Enumeration

Visiting the Web application, we are presented with a login page

![Website](/Medium_rooms/Recruit/Screenshots/01_Website.png)


Every time I see such a login form, I always test to see if there may be a SQL injection vulnerability. 


![SQL Injection attempt](/Medium_rooms/Recruit/Screenshots/02_SQLi-attempt.png)

I tried some basic SQL query injections, yet there is no sign of a SQL injection. No SQL errors, even trying common injections at the password field like `' OR1=1;--` didn't work. Instead, let's see if we can identify anything with the _Access API_ documentation

![API documentation](/Medium_rooms/Recruit/Screenshots/03_API.png)

That API endpoint looks like it could be vulnerable to SSRF (Server-side request Forgery). To test my thesis, I started a Python web server (`python3 -m http.server 8000`) and tried to see if the application would make a request to it. 


![SSRF Attempt](/Medium_rooms/Recruit/Screenshots/04_SSRF-attempt.png)
> I added `&x=` to the end of the URL in case the request looks different on the backend

Okay, that's sad! It didn't work. However this doesn't mean the endpoint isn't vulnerable. I have a high feeling that there is something to exploit here. I tried Path traversal as well with a common paths like `/../admin`, yet still no success

Off the broad, we didn't find anything, so that's why I started Gobuster to enumerate for hidden directories. 

```bash
gobuster dir -u http://recruit.thm -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -r -x.php,.txt
```

![Gobuster result](/Medium_rooms/Recruit/Screenshots/05_Gobuster-result.png)

To sum up: The most interesting directories here are `/mail`, `/assets`, as they have directory listing enabled. However `/dashboard.php`, redirects us to the login page and `/config.php` doesn't display anything, so that's also interesting. We can ignore `/phpmyadmin` as it's just a framework and we aren't trying to exploit that, we are only assessing the Web application

Surprisingly, the `/assets` doesn't have any valuable files, only JavaScript files which don't give any information about the behavior of the web app. 

The `/mail` directory has a `mail.log` file listed, opening it shows us an e-mail from the HR Team to the IT support, stating that the Recruitment Portal has been successfully deployed

![Email from HR](/Medium_rooms/Recruit/Screenshots/06_e-mail.png)

Alrighty, we have a valid username and know where the password for that account is stored, that's something! However, as I mentioned before, we can't actually see what's in `config.php`. I tried to retrieve it over the API endpoint, which didn't work

```txt
http://recruit.thm/file.php?cv=config.php

-> Only local files are allowed

# Trying with path traversal
http://recruit.thm/file.php?cv=/../config.php

-> Only local files are allowed
```

I must admit, I was stuck here for quite some while. I did some research and after a while I came across a vulnerability which was new to me until now. It's called **Local File Inclusion** or short **LFI**. It's pretty similar to SSRF I must admit. Instead of trying to make the application send requests to restricted pages and display it to you, you try to make the application access a local file on the system. Instead of the HTTP scheme, you use a scheme like `file://`. It's basically the same when you open a google PDF file (to name an example) on your system, it also accesses it with `file://`! 

Trying this logic with the `config.php` file, I got this:

![config.php file](/Medium_rooms/Recruit/Screenshots/07_Config.png)

Using our obtained credentials, we will log in and get access to the `/dashboard.php`

![Logged in as HR](/Medium_rooms/Recruit/Screenshots/08_HR.png)

Awesome! Now let's try to log in as admin. We have a search query here, which on it's own could point to several vulnerabilities. I first of all wanted to see how the Search function works. I searched for admin and noticed the URL changing to this:

```txt
http://recruit.thm/dashboard.php?search=admin
```

Yet again, I suspected a potential SQL injection. Again, I tried the most basic test, searching for `'` to see if we get an error.

![SQL injection identified](/Medium_rooms/Recruit/Screenshots/09_SQLi-n2.png)

Oh yeah, now this looks promising! These past days I've been looking to find a challenge where I need to exploit a SQL injection vulnerability, and I finally found one. While learning about SQL injections, I also learned about a tool called **sqlmap**, which I knew existed but honestly forgot about until like last week. This tool could actually help us out a lot here. 

I checked which kind of request the search query sends to the application in the Browser developer tools under network

![HTTP Method used](/Medium_rooms/Recruit/Screenshots/10_HTTP-method.png)

Normally, I'd use the `-u` flag in sqlmap for GET request, however since we need an authenticated session in order to send requests to this page, it's better we provide sqlmap with a requirement file, which just contains the package we are sending. 
For this, open Burp suite. I hope you are familiar with it, if not try to follow these brief instructions if you are on the AttackBox:

Open Burp suite -> Click on "next" until the main Interface opens -> Click on Proxy -> Intercept on -> On Firefox, click on the proxy at the top right corner -> activate Burp proxy -> sent a new request to the page -> Proxy should open -> Ctrl + R to send it to Repeater

In the end, it should look like this:

![Intercepted Request with burp](/Medium_rooms/Recruit/Screenshots/11_Burp.png)
> You might want to send a valid parameter value such as `a` instead of `'` 

Copy it to a file named `req.txt` and then, use the following command

```bash
sqlmap -r req.txt --dbs
```

![sqlmap](/Medium_rooms/Recruit/Screenshots/12_sqlmap1.png)

We are interested in the database `recruit_db`, so now let's retrieve it's tables

```bash
sqlmap -r req.txt -D recruit_db --tables
```

![sqlmap2](/Medium_rooms/Recruit/Screenshots/13_sqlmap2.png)

Since we are looking for admin credentials, we are more interested in the `users` table instead of the `candidates` ones. 

Now that we know from which table we want to retrieve data, our last command to sqlmap will be to dump all the columns in that table

```bash
sqlmap -r req.txt -D recruit_db -T users --columns --dump
```

![sqlmap3](/Medium_rooms/Recruit/Screenshots/14_sqlmap3.png)

Jackpot! For your final flag, just log in with these credentials and we are done, very successful!

## Conclusion

In this room, we had to exploit a LFI vulnerability in a vulnerable API endpoint to get access to the web page as HR. After that, we identified a SQL injection and successfully retrieved the database entries of the application to get admin access. This was my first time actually learning about a Local file inclusion, so I am very satisfied with my work on this challenge. Again, I try to explain as much as possible so people who may struggle with challenges (which I do quite often!) actually understand what was being done. I hope this write-up helped you out :D