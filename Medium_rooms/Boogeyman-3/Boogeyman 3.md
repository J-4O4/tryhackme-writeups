# Boogeyman 3 write-up

![Picture of the room](/Medium_rooms/Boogeyman-3/Screenshots/00_Room.png)

Oh noo!! The boogeyman is back again, and this time he is targeting the CEO! Such a pain in the neck! Let's do this one more time and scare him away for ever. Quick Logistics LLC hired a managed security service provider (MSSP) this time, so our job is to analyse the provided logs with our SIEM, Elastic stack (ELK).

**Check out my write-ups to the other rooms of the series:**
- [Boogeyman 1](/Medium_rooms/Boogeyman-1/Boogeyman%201.md)
- [Boogeyman 2](/Medium_rooms/Boogeyman-2/Boogeyman%202.md)


## Log investigation

The security team gave us initial information about what happened that might have compromised the CEO. They provided us that a double extension file from a phishing Email was downloaded, called **"ProjectFinancialSummary_Q3.pdf"**, which is actually a Disc Image file, and in the DVD Drive (D:) there's another file with the identical name but as a HTML file. Before we proceed and search for it in the logs, make sure to filter for the range  **August 29 and August 30, 2023** as that's the time the incident occurred  
![Right time range](/Medium_rooms/Boogeyman-3/Screenshots/01_Time-filter.png)
> ! I messed up the time filter for the first few questions, I eventually realized when certain event's weren't in the logs. Set the filter from the 29th of August 12am to 31th of August 12am instead! 

Now, we will search for Sysmon process execution Event ID 1 and for the file name to see if it is related to a process execution

```text
event.code: "1" AND *ProjectFinancialSummary_Q3.pdf*
```

![Query 1 result](/Medium_rooms/Boogeyman-3/Screenshots/02_Query-1.png)
> It may look different for you, since I selected some fields that are of interest. You can do the same on the left hand side, search for field names and click on the "+"

Okay, now that we know the process ID, it will be valuable to know which processes got created by this process, so instead of searching for the file name, we will search for events having a PPID of of the found process

```text
event.code : "1" AND  process.parent.pid : 6392
```

![Query 2 result](/Medium_rooms/Boogeyman-3/Screenshots/03_Query-2.png)

You'll notice some new files were created by two different processes which got created by the suspicious process. There's also a scheduled task being created with PowerShell called "review", judging by the **Register-ScheduledTask** field. This is likely a persistence mechanism.
By now we know the Boogeyman loves c2 connections, so it might be a good idea to check if any of the found processes have any network activity events, which is Sysmon event ID 3. It would be very suspicious if processes like rundll32.exe or powershell.exe generate such events

```text
event.code : "3" AND *rundll* OR *xcopy* OR *powershell*
```

![Query 3 result](/Medium_rooms/Boogeyman-3/Screenshots/04_Query-3.png)

That's a lot of hits, and all of them seem to reach for the same destination IP address and port. Since the CEO was in a local administrator group, this means high privileges for the attacker to abuse. He seemed to be bypassing UAC somehow, so I had to look up common bypass techniques. My research lead me to a trusted windows process called "**Fodhelper.exe**", and looking that up in the logs reveals the attacker used it and was caused by the malicious review.dat file which the attacker implemented. 

```text
event.code : "1" AND *Fodhelper.exe*
```

![Query 4 result](/Medium_rooms/Boogeyman-3/Screenshots/05_Query-4.png)

It was then when I found out my time filter was incorrect. That was a dumb mistake which costed me time and nerves. I just couldn't find out which tool the attacker downloaded for the data dumping. Anyways, after I updated it, I looked for PowerShell activity where GitHub got mentioned. I would've used Event code 15 for the full URL, but since the attacker can't use the browser of the compromised machine, it must've been through PowerShell.

```text
*powershell* AND *github.com*
```

![Query 5 result](/Medium_rooms/Boogeyman-3/Screenshots/06_Query-5.png)

Another mistake from mine was I only focused on sysmon event codes and not on the basic windows logs. Windows logs offer us some things that sysmon logs can't provide. I'll focus on the windows event ID used to log PowerShell commands, which is 4104. We also know the tool the attacker used, so let's see which other user he got

```text
event.code : 4104 AND *<tool_name>*
```
> Edit: It might be a better Idea to just search for the process name of the tool: `process.name : <tool_name>`

![Query 6 result](/Medium_rooms/Boogeyman-3/Screenshots/07_Query-6.png)

We notice the attacker first dumped the credentials of the administrator user, and afterwards of another user. Event code 4104 won't help us anymore, as the attacker might use his files to create new PowerShell processes which would be logged as process creation events, thus Sysmon event ID 1.  I created following query:

```text
process.name : "powershell.exe" AND event.code : *1*
```

I also added the "**process.command_line**" to my selected field and started scrolling to the bottom of the results and back up while paying attention to the commands. I finally got to understand a bit more of the attackers behavior and found some interesting commands 

1. The Attacker accessed a file from the File System and read it's content:
	![Filesystem file accessed](/Medium_rooms/Boogeyman-3/Screenshots/08_Access-file.png)
2. It looks like this file contained some sensitive data, as afterwards the attacker seems to have found new credentials 
	![New credentials](/Medium_rooms/Boogeyman-3/Screenshots/09_new-credentials.png)

This indicates that the attacker is now causing damage on another machine, so we will need to investigate what kind of commands the attacker used there. We will just add the name of the host he just got access to and add it to our query

```text
process.name : "powershell.exe" AND event.code : "1" AND *<host_name>*
```

![Malicious code on second host](/Medium_rooms/Boogeyman-3/Screenshots/10_2nd-host.png)
> The field on the right sight is the **process.parent.name** field which I added to the selected ones, as we wanted to know which parent process was used

Right after the attacker got access to the host, he ran a malicious command which is even base64 encoded. I decoded it out of curiosity and this is probably the command used to connect to his C2 server. On this host, he does the same as on the other host: Investigating the file system, installing his little tool to dump further credentials, and moves to the next host and leaves this one with installed ransomware. However, we don't know yet if the attacker actually got access to another machine, so let's check what his tool did this time

```text
process.name : "<tool_name>"  AND event.code : "1" AND *<host_name>*
```

![Malicious code on second host](/Medium_rooms/Boogeyman-3/Screenshots/11_dumped-credentials.png)

It looks like he only got access to the domain controller and performed a dcsync attack to dump further credentials. To investigate, we will look up what he dumped with that 

```text
process.name : "<tool_name>" AND "dcsync"
```

![DCSync attack](/Medium_rooms/Boogeyman-3/Screenshots/12_dcsync-attack.png)

Lastly, the attacker downloaded ransomware, which I already mentioned before. I actually found that out pretty early in my investigation. You will find the download as a PowerShell command, so we will be reusing my previous query: `process.name : "powershell.exe" AND event.code : "1"`

![](/Medium_rooms/Boogeyman-3/Screenshots/13_ransomware-download.png)

## Conclusion

I didn't expect this log analysis would be too challenging, but it definitely was! I must admit I didn't have much experience with ELK. My investigation probably wasn't the best as I did some dumb mistakes due to my inexperience which wasted crucial time. However, the room really teaches some valuable lessons, for example actually understanding the history of commands, what they caused, and taking notes on what you've found already. To sum up the behavior of the boogeyman, he likes to use spear phishing attacks in order to gain access to the system. He then proceeds to establish a connection with a C2 server to collect Data and remain persistence. This time he actually managed to move through the network with collected credentials and successfully downloaded ransomware. We caught him again so it is likely he will finally leave us for good

Thanks for reading my write-ups of the boogeyman series! 
