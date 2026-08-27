# Boogeyman 2 Write-up

![Picture of the room](/Medium_rooms/Boogeyman-2/Screenshots/00_Room.png)

SOS! the Boogeyman is back, this time with new tactics, techniques and procedures. The last time, the boogeyman targeted the Quick Logistics LLC fiance office. You can view my write up to the first boogeyman here: [Boogeyman 1](/Medium_rooms/Boogeyman-1) . This time, the HR office got targeted, and we've got a copy of the phishing email and a memory dump of the victims workstation to analyse what the Boogeyman caused now. 

**Check out my other Write-ups to the Boogeyman series here**:
- [Boogeyman 1](/Medium_rooms/Boogeyman-1)
- [Boogeyman 3](/Medium_rooms/Boogeyman-3)

## Email Analysis

The email which compromised the system appears to be using a spear phishing technique to get the victim to download the file. A normal person would probably not think this is a phishing email, as it appears very trust worthy

![Picture of the email](/Medium_rooms/Boogeyman-2/Screenshots/01_Email.png)

You'll see the email looks legit, the attachment looks legit, there aren't really any red flags. There may be some entries about this document on Virustotal if the attacker used the same persona for all of his victims and didn't change anything on the file. So let's generate a hash for this file. Save the attachment to your desktop or a path of your choosing, but as always, don't ever open it.

![Hash of the attachment](/Medium_rooms/Boogeyman-2/Screenshots/02_MD5-hash.png)

Looking it up on VirusTotal, we get to know it's a malicious file containing macros, even though it doesn't have a macro extension. You'd usually expect a `.docm` if there's a macro in a word document. Turns out this file is a `ole` file

![VirusTotal result](/Medium_rooms/Boogeyman-2/Screenshots/03_VirusTotal-result.png)

Only using the VirusTotal result wont bring us far enough, it may only reveal the related URL to download the stage 2 payload. If we really want some deeper insights, we can use the provided tool in this room, **olevba**, which scans the macros and it's behavior of a document file like `.doc`

![Macro behavior](/Medium_rooms/Boogeyman-2/Screenshots/04_Olevba-result.png)

## Memory analysis

Now that we know what the malicious attachment of the email does, we should analyse the artefact of the extracted memory from the victims host. After all, the host got compromised, so it's likely the attachment got executed. We'll use Volatility3 to analyse it. Let's build a process tree to see Which processes got executed

![Pstree of memory file](/Medium_rooms/Boogeyman-2/Screenshots/05_Memory-process-tree.png)
> The Numbers on the left side are the PID numbers, the ones following after that are the Parent PID numbers

looking at the results, you can notice a process was executed by the wscript.exe process, but where did it come from? We already know a URL which was used to download the stage 2 payload by the macro. Maybe we can find that URL again in the strings of the memory artefact. I chose to search for the domain name in the strings using following command:

```bash
strings WKSTN-2961.raw | grep "boogeymanisback"
```

![strings containing the domain](/Medium_rooms/Boogeyman-2/Screenshots/06_strings-result.png)

For further investigation, we must find out which process actually created the connection to the C2 server, and we can do that with the `netscan` plugin:

```bash
vol -f WKSTN-2961.raw windows.netscan | grep "updater.exe\|wscript.exe"
```

![established network connection](/Medium_rooms/Boogeyman-2/Screenshots/07_Network-connection.png)

Now we need to find out a path to the file that established a C2 connection. Since we are working with files now, it's a good idea to use the `filescan` plugin next and search for the path the found binary might hide

![Path of the binary that established C2 connection](/Medium_rooms/Boogeyman-2/Screenshots/08_Binary-path.png)
> It might take some time until the command is finished, and since we will run it again, I would recommend saving the output in a file and just use grep to search through that

Let's do the same with the email attachment from earlier, as this must be saved somewhere as well

![Path of the file from the email](/Medium_rooms/Boogeyman-2/Screenshots/09_File-from-email.png)

Last but not least, we got the information that a scheduled task has been created right after establishing the c2 callback. We might find something about that in the strings of the memory file, else we will do some deeper analysis on the suspicious binary **updater.exe** we've found

```bash
strings WKSTN-2961.raw | grep "schtasks"
```
> schtasks is the windows process which creates scheduled tasks. If you don't know these names, it's always one internet search away (I forgot the name as well ;P)

![Path of the file from the email](/Medium_rooms/Boogeyman-2/Screenshots/10_schtasks-activity.png)

In the first box, you can see the command of a scheduled task being created, and in the second box you see something which seems to be coming from the c2 connection, judging by the base64 encoded strings. In the second box, the attacker created another scheduled task to maintain persistence.

## Conclusion

For this room, I had to learn volatility in order to do the memory forensics, so if the second part of the write-up felt confusing I am sorry. I still tried my best to make it as understanding as possible! I also had fun completing this room as it was something new. Memory forensics is fun! Anyways, we caught the boogeyman yet again, hopefully he will move on to his next target and leave us alone.... oh.. [Boogeyman 3](/Medium_rooms/Boogeyman-3)
