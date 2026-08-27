
# Boogeyman 1 Write-up

![Picture of the room](/Medium_rooms/Boogeyman-1/Screenshots/00_Room.png)

Boo! The Boogeyman is here! This room is part of the 3 part long Boogeyman series from the SOC Level 1 Capstone Challenges module. I am looking forward to write a write-up for each of them. Now let's scare the boogeyman away, shall we?

**You can find the Write-ups to the other rooms once I am done with them:**
- [Boogeyman 2](/Medium_rooms/Boogeyman-2/Boogeyman%202.md)
- [Boogeyman 3](/Medium_rooms/Boogeyman-3/Boogeyman%203.md)

## Email Analysis

Let's gather information about how the boogeyman get's his victims. The fiance team is the one that got targeted with looking to be personalized phishing emails. Since we have the email as an artefact, we can open the .eml file with Thunderbird on the system to gather some basic information like the e-mail from the sender and recipient 

![The phishing mail](/Medium_rooms/Boogeyman-1/Screenshots/01_Phishing-mail.png)
> I had to redact a lot as it answers some of the questions of this task

As you can see the e-mail looks very legitimate at first glance. It's a good idea to look at the email source to find out more about the sender, like the IP address. We could theoretically look up the IP on Cyber threat intelligence websites to get an idea where the actor is located. However, for this task we need to specifically look at the **DKIM-Signature** and **List-Unsubscribe** headers, so let's open that terminal!

![DKIM header values](/Medium_rooms/Boogeyman-1/Screenshots/02_Email-header.png)

I hope you still have Thunderbird open so we can save that attachment to our desktop. Note that password from the email as we will be needing that now

![Archive unziped](/Medium_rooms/Boogeyman-1/Screenshots/03_unziped-archive.png)

We'll use the **lnkparse** and look out for the Command line arguments field

![lnk parsed](/Medium_rooms/Boogeyman-1/Screenshots/04_lnk-file-command.png)


## Endpoint investigation

For this task, you should use the provided JQ cheatsheet to parse through the PowerShell history. I recommend trying the provided commands before investigating the history to get a feeling of this tool, that's what I did. After I understood the commands, I used the `cat powershell.json | jq` command to take a look at a sample log entry. One Field which caught my eye was the `ScriptBlockText` field, as it shows the executed command. I proceeded with sorting the events by time and only view the **ScriptBlockText** field

![Viewing PowerShell history](/Medium_rooms/Boogeyman-1/Screenshots/05_PowerShell-history.png)

We can see the threat actor starting his PowerShell history connecting to his c2 server. If we continue to scroll down the ordered history we will notice that the threat actor downloaded something from GitHub

![Tool download](/Medium_rooms/Boogeyman-1/Screenshots/06_downloaded-tool.png)

The room provided us with the name of a binary that was used. We would've found it ourselves if we were continuing to look through the history. To make things easier, let's find every command where this binary got mentioned

![Binary used](/Medium_rooms/Boogeyman-1/Screenshots/07_used-binary.png)

Pay attention! This one got me too! The binary is already being ran in a path which is unbeknownst to us. There's two approaches you could take here. One would be to look at the activity that happened before the binary was executed to see when the path got changed, or search for the Path with `grep` ! Since it's being ran on an AppData path, this suggest it got executed from a user. All users are located at `C:\Users`. You see where this is going?

![Starting path](/Medium_rooms/Boogeyman-1/Screenshots/08_starting-path.png)
> You can craft the full path with the blue marked strings, for that one question if you struggled with it

Actually, this command gave us something interesting as well. If you look closely, there was an data exfiltration attempt made on a `.kdbx` file. Even the name of that file indicates it might be sensitive data, and when doing a small research on what this extension means, you'll quickly find out it's a Password database called `REDACTED` 

We'll need to wander through the PowerShell history again, there may be some other information we should  take notes of. In the PowerShell history, after the exfiltration event, you can catch an encoding command which shows how the data has been exfiltrated

![Data encoding](/Medium_rooms/Boogeyman-1/Screenshots/09_data-encoding.png)

## Network Traffic Analysis

Alrighty, let's hunt down the final traces of the boogeyman. Boot up Wireshark and view the packet capture. In our PowerShell logs, there was an IP address of the file sharing host. We can write a simple Wireshark query like

```Wireshark
http && ip.dst_host =="167.71.211.113"
```

Follow the http stream for information about the response from the server, which will expose which tool got used. 

![HTTP stream files host](/Medium_rooms/Boogeyman-1/Screenshots/10_files-host-tool.png)

For the C2 server, we only know the domain, but that's enough: `http contains "cdn.bpakcaging.xyz"`. I guess if you are experienced with PowerShell, you might already have an idea of what's happening on the c2 server. However, you will quickly notice some POST requests in the capture file, some bigger than the others. It seems like the output got transferred as decimal values. 

![c2 output](/Medium_rooms/Boogeyman-1/Screenshots/11_c2-output.png)

To find more about the results of the commands, we will add a filter for Post methods to our filter: `http.request.method == "POST"`
The full filter:
```text
http contains "cdn.bpakcaging.xyz" && http.request.method == "POST"
```

With that, all we need to do is focus on some bigger POST request and decrypt them to text, this way we know which sensitive data got revealed. With bigger ones, I mean every package with a length over 1000. (To safe you some time, check out package 44467)

![Decrypted Post method](/Medium_rooms/Boogeyman-1/Screenshots/12_leaked-password.png)

I would've called it a day by now, but remember there is still DNS activity from the data exfiltration. However, there are over 400 packages when filtering for the IP of the file hosting site and for DNS. I'm afraid I have to use Tshark, but I never used it before, so I had to get a litte help from that little echo assistant. After some tedious back and forth with that assistent, I managed to get this query out of him. (Note to myself, I should definitely study Tshark next)

```text
tshark -r capture.pcapng -n -T fields -e dns.qry.name | grep "bpakcaging.xyz" | cut -f 1 -d "." | uniq
```

I saved the results in a file and when I tried to decode the result which is obviously full of hex strings, as that's what the attacker used, I got a completely gibberish result. Turns out that's okay, as the attacker transported the file through many DNS queries and crafted it together afterwards instead of transporting the whole file. Kdbx files have binary data like .exe files, so the gibberish from decoding is expected. However I still need to remove the newline characters (\n)

```text
cat result.txt | tr -d '\n' > cleaned.txt
```
> I saved the result of the first command in `results.txt` and the result without the newline characters in `cleaned.txt`

Just concatenate the cleaned file and copy the content in there, go to CyberChef and use the "from hex" recipe and save the output as a `.kdbx` file. Now all you need to do is to open that file and enter the password from the previous question

## Conclusion

We hunted down the boogeyman, congrats! They were targeting fiance groups with personalized emails, established a connection to their c2 sever and exfiltrate valuable data like credit card numbers. Now go ahead, you never know when the boogeyman will strike again -> [Boogeyman 2](/Medium_rooms/Boogeyman-2/Boogeyman%202.md)


