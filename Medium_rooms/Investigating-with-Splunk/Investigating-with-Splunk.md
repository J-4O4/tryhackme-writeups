# Investigating with Splunk write-up

![Picture of room](/Medium_rooms/Investigating-with-Splunk/Screenshots/00_Investigating-with-Splunk.png)

In the Investigating with Splunk challenge, we play the role of an SOC Analyst and our objective is to investigate the logs and identify anomalies. I've been studying Splunk for the past days, so let's put my skills to the test. This is my second write-up so far, so I hope you will be able to follow along!

## Question 1
Before we can start, we need to visit the splunk interface with the specific logs, we can do this by opening our browser: http://<TARGET_IP>
Once we reached Splunk, we will click on "Search and Reporting" on the top left corner underneath "Apps". The room mentions that the logs to investigate are ingested in the index "main", so we will run the following query to see all of the events in index main, and don't forget to set the search filter on "All time"

```bash
index="main"
```
![Picture of search result](/Medium_rooms/Investigating-with-Splunk/Screenshots/01_First_query.png)
With that, the first question of the room is answered

## Question 2
For this question, we need to figure out if the attacker was successful in creating a backdoor user on the system. First of all, look at the left side of the search page, where all our fields are listed. Let's look at the selected fields and check which sourcetype is mentioned. The only sourcetype available there is: "event_logs", so we are probably investigating Windows event viewer logs. Since it's the only sourcetype avaiable in our logs with the index 'main', we won't need to add that to our query, instead, we should look for other interesting fields, to find one that includes the Windows Security Event IDs... Found it! The field is called EventID. Now all we need to know is which event ID is corresponding to Account creation. I looked it up, it's ID 4720. With that, we can build our query:

```bash
index="main" EventID="4720"
```
And there is our answer:

![Picture of search result](/Medium_rooms/Investigating-with-Splunk/Screenshots/02_New_User.png)

## Question 3
For this question, I firstly tried to look if there is an event ID which could indicate that a registry key was updated. I checked if ID 4657 was in the logs, which indicates that a registry key was changed or updated, but there were no results. I then added the hostname 'Micheal.Beaven' to my search, so I only get results for this specific host. After that I checked the fields of interest to see if there is anything that could help me, and indeed there was! Field "Category" included a value with "Registry value set (rule: RegistryEvent)". I clicked on it to add it to my query, and then there were only 10 events left. With that, I found out that there is an Event ID for that all along, which turned out to be Microsoft Sysmon Event ID 13. Now all I had to do is check the "TargetObject" field to see which path includes User REDACTED. The following queries work:

```bash
index="main" EventID="13" Hostname="Micheal.Beaven"
OR
index="main" Hostname="Micheal.Beaven" Category="Registry value set (rule: RegistryEvent)"
```
![Picture of search result](/Medium_rooms/Investigating-with-Splunk/Screenshots/03_Registry_Path.png)

## Question 4
Well, this question kinda confused me. We already knew the user name of the backdoor account. I tried to dig deeper into the logs, but the answer was directly in front of my face from the beginning. I tried to enter Users like micheal, since it was the name of the host that created the backdoor user. However, you didn't really need to log deep into the logs, nor write a search query. On the selected fields, you just had to look under the user field to see if there is a username similar to the backdoor account, this is the account the attacker wanted to impersonate with his backdoor account, thus the answer to question 4. If you wanted to see every user mentioned in the logs, I found a field called "SubjectUserName", just search it under the fields of interest. 

## Question 5
Surprisingly, this question was easier than question 4 for me, because it wasn't as confusing to me. The only hard part was to figure out where to start. Up until now, every Question except for 4 had some EventID involved. Remote execution usually involves creating a new process on the target system, you first need to create a process on the machine to do so, for example a PowerShell process. I researched which ID represents a Process creation and found ID 4688. So I added that to my query. There was an interesting field called CommandLine, so I knew we have to include this in our search. For this task, I wanted to create a table to see every value in the commandline Field which includes the name of the backdoor account "-1-----" . I quickly checked my notes to see how I could make that table and here is my final query:
```bash
index="main" EventID="4688" -1-----
| table _time CommandLine
```

![Picture of search result](/Medium_rooms/Investigating-with-Splunk/Screenshots/04_Table_result.png)

We can see there are 3 events in our table, the first one is the command we were looking for!

## Question 6
Tricky question! Again, I looked for event IDs which stand for Login attempts. These are 4624 for a successful one and 4625 for a failed one. So I ran following query to look for both which came from our Backdoor user
```bash
index="main" (EventID="4624" OR EventID="4625") -1-----
```
But there were no Events!? Did I do something one?? Well, turns out that's the answer. I'll just leave it by that... took me 5 minutes until I thought that this could actually be the answer...

## Question 7
Finally, a straightforward question again. Simply just add the Backdoor account to your query again, then look at the Channel field which includes 4 values, click on the Windows Powershell one to view the event it returns. Once there, you can see the look at the hostname field to see which host was likely infected. I even looked at the EventID to see how you would've found that with the EventID. Since I started to like the table function, I created a table for visualization purposes:

![Picture of search result](/Medium_rooms/Investigating-with-Splunk/Screenshots/05_TableForQ7.png)

## Question 8
Another easy question, in my opinion. My first approach was a quick research again to see if there is an Event ID for logging Powershell. I got 2 results, 4103 and 4104. Then All I did was to enter a pretty easy query

```bash
index="main" (EventID="4103" OR EventID="4104")
```

And all you need to do is check how many events are listed when you hit enter!

## Final Question
We'll stay with our query from previous question as we need a specific powershell Script. Finding that encoded Powershell script isn't that hard, just look at one of the events under ContextInfo and look for Host Application = ...
From that, we only need the encoded string from that command. The hard part is to actually decode it. It is likely a Base64 string, I could tell by the two equal signs at the end of the string. I pasted it to [Online Base64 Decoder](https://www.base64decode.org/)
but the output was just gibberish. I then took a look at the hint, which didn't help much as it just suggested to defang the URL with cyberchef instead of actually telling me how to decode the string in the first place. Let's keep that in mind though. Sadly I wasn't all familiar with base 64 decoding, so I had to ask an AI how to decode it correctly without spoiling everything to me. It told me something about "Base 64 -> UTF-16LE" which just confused me even more. However, on that website I mentioned, I noticed there is an option for "Source Character Set" being set to UTF-8, and you could actually change it. I found UTF-16LE there and tried to decode it again

![Picture of search result](/Medium_rooms/Investigating-with-Splunk/Screenshots/06_decoded_string.png)

I took a closer look on the decoded string and saw something interesting (Which is marked on the picture above). There is another Base64 String in that code, which likely contains the URL. before we decode that, keep in mind this URL ends with /news[.]php looking at that marked line. We will decode that Base64 string on the same website again with UTF-16LE. Now, let's switch to [CyberChef](https://gchq.github.io/CyberChef/) to defang that URL we got to answer the last question

![Picture of search result](/Medium_rooms/Investigating-with-Splunk/Screenshots/07_defanged_URL.png)

```text
hxxp[://]10[.]10[.]10[.]5/news[.]php
```

### Finally!

## Conclusion
Well, that was difficult, it took me way longer than the expected 30min. I definitely need more practice with Splunk to investigate logs more efficiently and study some Windows Event IDs. I'll try to do easier splunk challenges for now. I know this write-up isn't the cleanest, but hopefully it was helpful! Again I learned some interesting things, so I am excited to do more write-ups in the future!
