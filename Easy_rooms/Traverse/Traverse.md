
# Traverse Write-up

![Picture of the room](/Easy_rooms/Traverse/Screenshots/00_Room.png)

I am back with another write-up! I've been studying the contents of the Security Engineer file recently and this Challenge is part of it. So why not create another Write-up :D.  Looks like Bob didn't do such a good job as a security Engineer at his firm. Let's investigate what went wrong!

## Analyzing the Webpage

Once we open the Webpage, we are greeted with a very welcoming message...

![Picture of the room](/Easy_rooms/Traverse/Screenshots/01_Mainpage.png)

There isn't really anything else going on when visiting the homepage. Usually, you want to analyze the Source of the web page to see if there is anything left which might be valuable. As soon as you open the source, you will notice there are quite a lot of comments 

![Picture of the Websource](/Easy_rooms/Traverse/Screenshots/02_Websource.png)

As you can see, the comments reveal some things they probably shouldn't. We can't really tell if Bob was the one who left it there or if it was the threat actor, but we will find that out later eventually. You'll spot a custom JavaScript file in the Website's header tag. I'd suggest we look into that first before we will visit the Paths mentioned in the other comments. 

![Picture of the HEX encoded code](/Easy_rooms/Traverse/Screenshots/03_Custom.png)

Okay, this is obviously Hex encoded. To decode it, we will need to convert to ASCII. Since I've been coding a lot recently, I could probably write a program to do it, but we don't want to make it unnecessarily complicated. An online Hex to ASCII decoder will do the job.

![Picture of the decoded HEX code](/Easy_rooms/Traverse/Screenshots/04_decoded.png)
> For the Answer in the room, build a sentence out of the capitalized words. 

If this code would actually do something, we should try to understand it. However, since this code does absolutely nothing, we can move on to visiting the hidden paths. `/img` shows nothing of importance, as it just contains the pictures of the site. `/logs` on the other hand should immediately catch your eye. You do never ever want to expose your logs to the internet. 

![Picture of the /logs path](/Easy_rooms/Traverse/Screenshots/05_Logpath.png)

There's a text file, let's click on it to view it's content

![Picture of the text file](/Easy_rooms/Traverse/Screenshots/06_Email.png)

Looks like Bob was in a hurry for his holidays, who can't relate. This might be a reason why the Website got hacked at the end. Security however is a really important factor and shouldn't be rushed! Anyways, Bob left us with a quiz time: What's the first Phase of the (S)SDLC? If you've been enrolled in the security Engineer path like me, it's really obvious. The correct Answer is Planning, which will be the path to the API documentation (`/planning`)

Once there, we are prompted to enter a key to access the page. This will be the one Bob has provided in his email which I had to redact. After we entered the key, we can access the API documentation

![Picture of the API Documentaion](/Easy_rooms/Traverse/Screenshots/07_API-Docs.png)

## Exploiting the Webpage

That's a powerful API if you couldn't tell, yet sadly it doesn't have any kind of authentication nor authorization, which means anyone can use it. Seems like Bob was so excited for his holidays, he forgot to shift left. We'll be using the API to find an account which has an admin role. The room had a question where we had to look up the user with id=5, but that was just a normal user. After trying a bit of ids (Admin usually have a low id number), I found one:

![Picture of an Admin Account](/Easy_rooms/Traverse/Screenshots/08_Admin.png)

Perfect, this will enable us to do some damage if we wanted to exploit the page. Since we don't do that and instead are helping Bob with his hacker problem, we will only try to log in there to validate the credentials being exposed. Visit the path to the LoginURL for admins. We will need to authenticate using the email and password credentials which we found with the API query. Once logged in, you are presented this nifty page

![Picture of the Admin page](/Easy_rooms/Traverse/Screenshots/09_Adminpage.png)

The provided Commands are corresponding to the `whoami` command and `pwd` command. There isn't a way for us to enter commands like ls or anything like that to discover more about the Server. But if there is a will, there's a way. If we open our developer tools and go to the Network tab (`Right click -> Inspect -> Network`) and resend the data, you'll notice a POST request being made. Inspecting the POST requests body, one might see that there is a command field with the corresponding command. This must mean we can edit it (Right click the POST request and click `Edit and resend`) to send every command we would like (Even a reverse Shell for example, but we aren't exploiting today!). 

![Picture of the POST request](/Easy_rooms/Traverse/Screenshots/10_POST-request.png)
> In our case, change `whoami` to `ls` to view which files are currently on the server


![Picture of the POST request](/Easy_rooms/Traverse/Screenshots/11_POST-result.png)

There are files on the server that shouldn't be there if you couldn't tell. I wouldn't recommend visiting the web shell on the server, instead we should visit the `renamed_file_manager.php`. Keep our current Directory in mind, as we need to visit it from our current directory

Another log in, but for some stupid reason the password for this page is actually listed with the ls command. Luckily for us though, this means we can login and have full access to the whole website. 

![Picture of the File Manager](/Easy_rooms/Traverse/Screenshots/12_File-manager.png)

Lastly, let's clean up the mess the threat actor made by editing `index.php`. We will change the `$message` variable to something else so our flag will be the new message on the Webpage 

![Picture of index.php](/Easy_rooms/Traverse/Screenshots/13_Cleaned-Up.png)

## Conclusion

Well, that was quick and easy! I liked this room as you really had to use the knowledge you've gained through previous rooms in the Security Engineer path. In my opinion, this is a really good example of how multi-stage exploits work and what harm they might cause. I also liked that you always had to follow a different track! As always, I hope this write-up helped you out!  