# Snapped Phish-ing Line Write-up



![Picture of the room](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/00_The-Room.png)



Another room focusing on phishing. In my previous write-up, I covered the Greenholt Phish challenge. This room is about phishing analysis and adversary behavior. 



## Email investigation



I'll start by opening the phish-emails folder and open the email named "Quote for Services Rendered processed on June 29 2020 100132 AM.eml". Once the email opened, we can read the name of the individual who received the email, as well as the email address used by the attacker.



![email header information](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/01_email-header.png)



Next, open the email addressed to Zoe Ducan, it's the one next to the email we just opened. Let's download the attachment "Direct Credit Advice.html" from the Email. Although we are working in an isolated lab, I will still rename the file to "Direct-Credit-Advice.html" instead of opening it, so I can view the file contents through my terminal. Immediately, the URL in the header section of the HTML code caught my eye. 





![redirecting URL](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/02_redirecting-URL.png)



However, we are now forced to open the page to figure out which brand the login page is trying to impersonate



![Fake login page](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/03_login-page.png)



Once we've done that, we will navigate to the /data directory of the site. Luckily, the attacker exposes valuable information in that directory. There, we should look for the archive file, which is likely the ZIP archive

![Data directory](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/04_Data-Directory.png)



Download that zip file, open your terminal, and enter the following commands to calculate the hash:



```bash
cd Downloads
sha256sum ...365.zip
```


![Calculated hash](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/05_Hash-calculated.png)



## File Analysis using VirusTotal



After that, we analyze the hash on [VirusTotal](https://www.virustotal.com/gui/home/search). Enter the hash to see the VirusTotal results. There, all we have to do is look for the "Threat categories" found at the "DETECTION" tab



![Threat category](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/06_VirusTotal-Detection.png)



Continue by moving to the "DETAILS" tab of VirusTotal, scroll down a Tiny bit and look out for the "Contained Files" field under "Contents Metadata"



![Contained Files](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/07_VirusTotal-Details.png)



## Further fake Login page investigation



Going back to the domain the Adversary used and navigating to the /data directory again, you will notice there is more than just the zip file in that directory, there is another directory called "Update365", so that's why we need to navigate there... Jackpot! A log.txt is in that directory containing really valuable information since it's log data. We can see everyone who fell victim to the fake login page in there. If you opened it, you just need to discover which email got entered twice! 



At this point, we can extract the .zip file we discovered earlier since we are on a lab machine. Navigate to your Downloads folder, extract the zip file. The submit.php file we are trying to locate is located at: 



```text
Update365 -> office365 -> Validation -> submit.php
```


I moved the file to my Desktop, opened a terminal and used grep to detect the email address. 



![Detecting the email](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/08_Email-detected.png)



## Finding the Flag



I returned to the URL from the HTML file found at the beginning of the room. I removed the gibberish string which brought you to the fake login page and tried manually accessing /flag.txt on the URL. This reveals the hidden .txt file, which contains a base64 encoded string. 



![Hidden flag.txt](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/09_Secret-text.png)



Finally, we decode the base64 string. I used [CyberChef](https://gchq.github.io/CyberChef/) as the Task suggested, which turned out to be more helpful than just a base64 decode website. When you decode the Base64 string, the flag will be written out in reversed format, this is why CyberChef is useful for this task, as you can reverse the characters of your input with the Reverse operation.



![Discovered Flag](/Easy_rooms/Snapped_Phish-ing_Line/Screenshots/10_The-Flag.png)



## Conclusion

Yet another fun challenge and and fairly easy, you really get to practice the skills you've learned in the Phishing Analysis module! A great way to wrap up the module! I hope this write-up was helpful if you had some problems! 

