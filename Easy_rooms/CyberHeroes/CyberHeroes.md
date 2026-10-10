
# CyberHeroes Write-up

![Picture of the room](/Easy_rooms/CyberHeroes/Screenshots/00_Room.png)


Im bored! Let's bypass some authentication! I wanna be part of the **_elite club of CyberHeroes_**, so let's find a way in there!

![The Website](/Easy_rooms/CyberHeroes/Screenshots/01_Website.png)

There is a login page, of course this will be our way in! 

![The Login page](/Easy_rooms/CyberHeroes/Screenshots/02_Login-page.png)

From here on, we can try to perform a lot a of stuff to get access. Every time I see a login form, I always ask myself "I wonder if it is vulnerable to SQL injection". To test this, I try following approaches:

- I first of all try entering `'` or `"` to see if any SQL errors are displayed by the page which would confirm the vulnerability
- If there are no SQL errors, you shouldn't just assume it's not vulnerable to SQLi, after all, SQL injections are one of the most commonly abused vulnerabilities. if the first approach failed, I try to comment the SQL Injection out by adding `;--` to the query
- If there's still no indicator, I try to create my own SQL queries and inject it. On the password field, I would try entering the following: `abc' OR1=1;--` 

These are the most common approaches, of course there are more but you should try these first and later try to get more into depth if you suspect something. After all, you don't rush immediately to exploit, you first gather general information. 

![SQL injection attempt](/Easy_rooms/CyberHeroes/Screenshots/03_SQLi-attempt.png)

Of the broad, SQL injection wasn't possible. Since we only performed pretty basic injections, we can't really assume there is no SQL injection possible. 
However, if you haven't already noticed, this is a `.html` page and not a `.php` login page. It could very likely be that the whole authentication logic is written in the source code! Or maybe there is a JavaScript that is getting loaded with this page. Another thing that supports this theory is that the page doesn't even update when entering credentials, the website just prompts an alert. 

![Source code](/Easy_rooms/CyberHeroes/Screenshots/04_Source-code.png)

well well well, if you would look at that! I must admit, I suck at JavaScript, but everyone who coded once in their life would be able to figure out what this script does. Seems like the elite club of CyberHeroes isn't that cool afterall. 

To sum up what the code does: There are hard coded credentials, not even encrypted! The Password is written in there as a reversed string, which isn't really challenging to reverse engineer. If both username and password are correct, the script will make a request to get a `.txt` file containing a flag. 

Let's test that out. If you are lazy (like I am), just use [CyberChef](https://gchq.github.io/CyberChef/#recipe=Reverse('Character')) and use the "reverse" recipe, so you don't need to type the reversed password in manually 

![Source code](/Easy_rooms/CyberHeroes/Screenshots/05_Flag.png)

And that's it! 

## Conclusion

Honestly that was way to easy! Pretty short Write-Up this time, so I tried to explain the basics of SQL injection as it's always the first thing I try when I am presented a Login form! I hope this Write-Up helped you out! 