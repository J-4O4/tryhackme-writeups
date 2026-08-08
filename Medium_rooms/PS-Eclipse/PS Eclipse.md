# PS Eclipse Write-up

![Picture of the room](/Medium_rooms/PS-Eclipse/Screenshots/00_Room.png)

Welcome to next write-up, this time I'll be tackling the PS-Eclipse room, where we will use Splunk to investigate ransomware activity. I've studied a bit more about Splunk, so I am excited to see how I improved! We will connect to our Splunk instance and without further ado, let's get started!

## Gathering Information About the binary

In order to identify what caused the ransomware activity, we need to discover the suspicious binary running on the endpoint. Let me craft a query where we can see every Image field containing an executable file

```text
index =* | regex Image = "\.exe$"  | fields Image EventCode
```

There are 100+ entries for the Image field. Even though we could identify the malicious looking binary there as it has a high amount of activity, I would suggest adding the EventCode field to the query and setting its value to 3, as this will show us executables with network activity, which is expected for ransomware. 

```text
index =*  EventCode = 3 | regex Image = "\.exe$"  | fields Image EventCode
```

![Suspicious Binary](/Medium_rooms/PS-Eclipse/Screenshots/01_The-binary.png)

And that's it without a doubt! Now we need to figure out from which address the binary was downloaded, and not gonna lie, I didn't like this task. Since it's an address and they expect you to enter a http defanged address as the answer, my first instinct was to look for web activity by appending strings to my query like `"http"` `"https"` `"GET"`. I found some addresses, but none were right. I even tried to look for PowerShell activity and filtering for commands like `Invoke-WebRequest` but there was just nothing useful, except for a Base 64 encoded string... which, when decoded, actually contained that address in a wget request… (I used [base64 decoder](https://www.base64decode.org/))

![decoded Base64](/Medium_rooms/PS-Eclipse/Screenshots/02_Decoded-String.png)

I defanged it with [CyberChef](https://gchq.github.io/CyberChef/) and then just out of curiosity, I searched for that address what other options there were to find it. Turns out it is found in some dns Events, as a value in the QueryName field, so the final Query I would run for this task is:

```text
index =* QueryName ngrok*
```
> This will only give 9 Events, and the right one is the one starting with 88..

## PowerShell activity 

Now, we know it was probably installed with PowerShell, judging by the executed command written in an encrypted base64 string. Finding the full PowerShell path is straightforward.:

```text
index =* powershell | regex Image = "\.exe$" | top limit=3 Image
```

![Powershell version](/Medium_rooms/PS-Eclipse/Screenshots/03_PowerShell-version.png)

> I wanted to keep the results visualized and keeping the results small, so the top command is just optional

Moving forward, we will keep investigating PowerShell activity to find out how the binary was executed with elevated privileges. I created a table using the CommandLine field which is part of PowerShell related events. I filtered the events to only show values of the field with the name of the binary. You don't need to create a table like I did, I just like it as it is more visually appealing, but make sure you include the `reverse` command to better understand the history

```text
index =* CommandLine OUTSTANDING_GUTTER.exe | table _time CommandLine | reverse
```

![Powershell version](/Medium_rooms/PS-Eclipse/Screenshots/04_Powershell-elevated-privileges.png)

We know from that query that the binary will run as the SYSTEM user. This section of the command gives it away `/RU /SYSTEM`. It will use the command `"C:\Windows\system32\schtasks.exe" /Run /TN OUTSTANDING_GUTTER.exe` to run the binary. To figure out the full name of the SYSTEM user on the machine, all I did was include SYSTEM in my query and look at the user field to see the full name

```text
index =*  SYSTEM | fields User
```
> You now just have to Format it right, `… \SYSTEM;"C:\Windows\system32\schtasks.exe" /Run /TN OUTSTANDING_GUTTER.exe`

Recalling from the time we searched for the address the file was downloaded from, we will remember there were 2 addresses pretty identical to each other. So I will just assume the Connection to their remote Server is the other address we found

 ```text
index =* QueryName ngrok*
```

Moreover, there was a PowerShell script installed on the same Location of the binary. We know the Binary is located at `C:\Windows\Temp\` and finding it isn't that hard either. A PowerShell script has the extension of `.ps1` , so that's all we need to locate. Just include the path of the folder and the extension to your query and the first result is the one we are trying to locate.

```text
index =* ps1 "C:\\Windows\\Temp\\"
```

![Downloaded script](/Medium_rooms/PS-Eclipse/Screenshots/05_Downloaded-script.png)

## Further Malware Analysis
To figure out more about the Binary and ransomware attack, let's search their hashes up on [VirusTotal](https://www.virustotal.com/gui/home/upload). Although for this task, we only need the hash of the PowerShell script, I was just curious to see what the VirusTotal result was for the suspicious binary. (Make sure to copy the SHA-256 value!)

![Name of the script](/Medium_rooms/PS-Eclipse/Screenshots/06_Script-name.png)

> You can view this information under the Details tab after entering the hash

Figuring out the ransomware note is a piece of cake! Just mentioning `.txt` in your query will be enough to find it. If it wouldn't have been enough, we could've added the Name of the Script without the extension to our query, Because Ransomware attacks are likely to include their name in their note.

```text
index =* .txt
```

![Note of the Ransomware](/Medium_rooms/PS-Eclipse/Screenshots/07_Ransom-note.png)

Lastly, we want to find out to which image file the user's desktop wallpaper was changed to. We will apply the same logic we used to find the ransom note, commonly used image extensions on windows are `.png` or `.jpg`

```text
index =* .png OR .jpg
```
> Here I am just searching for the strings in the events, not the actual image!

![Ransomware Background](/Medium_rooms/PS-Eclipse/Screenshots/08_Background-img.png)

Easy as that, we are done!

## Conclusion
Concludingly, this room is great to actually get a feel for a real investigation case while being fun to follow. Also a really quick but important challenge! Instead of just searching some values using queries, I actually felt like investigating a ransomware case! Hopefully this write-up was easy to follow and helped you out! 
