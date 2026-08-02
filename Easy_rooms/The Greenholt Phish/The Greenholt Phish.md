# The Greenholt Phish Write-up
![Picture of the room](/Easy_rooms/The Greenholt Phish/Screenshots/00_The-Room.png)

Welcome to a new Write-up. This room will require us to perform an analysis on a suspicious email as a SOC Analyst to identify if it's part of a phishing attempt. I recently learned a lot about phishing analysis, so this room is perfect for practicing what I learned. All of the answers to the questions are likely to be found on the lab machine the room provides or Websites

We will be starting with opening the challenge.eml file on the desktop and click on View -> Message Source

![Viewing Message Source](/Easy_rooms/The Greenholt Phish/Screenshots/01_Message-source.png)

With that, the email header and Message Source will be visible to us. Since it contains a lot of strings and data, I'd suggest saving the source in a text file to find the contents you are looking for with grep command. Luckily though, the Lab machine uses a notepad where we can search for key words within it

To search for what you are looking for, you click on "Edit" -> "Find" and search for something in the notepad where your search term is found

![Example for Searching](/Easy_rooms/The Greenholt Phish/Screenshots/02_Sample-search.png)

You automatically jump to the section where your input is found. We searched for the Transfer Reference Number, mentioned in the subject, thus it made us jump to the header metadata which is also viewable on a lot of email clients, being the section with headers like "Subject" "From" "To" and so on. This means you don't really have to search for something for the next questions since the answers are already on your screen.

## Deeper message Source investigation
Oh well, the room expected you to answer the first few questions without looking at the message source and just use the data shown by the Email client. Searching for the originating-IP Header wont help as it is redacted. However, we might find something from the Received Headers

![Finding Originating IP](/Easy_rooms/The Greenholt Phish/Screenshots/03_Originating-IP.png)

We found 2 IPs: 10.197.41.148 and REDACTED. You could tell we are looking for the REDACTED IP Address because it has a HELO associated with the domain from the Sender's Email: mutawamarine[.]com  

Now we need to look up who the owner of the IP is. We wont be successful identifying that in the Email Source, so we will use an IP Lookup tool. I will be using [Cisco Talos](https://talosintelligence.com/) but feel free to use any other IP look up Tool

![Cisco Talos Result](/Easy_rooms/The Greenholt Phish/Screenshots/04_Network-owner.png)

Going further, we are told to perform a SPF record check on the Return path. SPF stands for Sender Policy Framework, used to authenticate the Server of a Sender for an Email Address through a SPF Record, to put it simply. The Return-Path is most likely to be the domain of the sender, however, it wont hurt looking it up in the Email Source

Again we will be using an online tool, I don't know any SPF Record check tools, so I looked some up. I tried some and [MX toolbox](https://mxtoolbox.com/spf.aspx) did the job!

![SPF Record](/Easy_rooms/The Greenholt Phish/Screenshots/05_SPF-Record.png)

We will continue with a DMARC lookup. DMARC is a security control to secure Emails using both SPF and DKIM, to authenticate the Server and check the Signature of the Email. And you guessed it, we will need another online tool. This time, I found out MX Toolbox has also a DMARC lookup option. You can perform the check [here](https://mxtoolbox.com/dmarc.aspx)

![DMARC Record](/Easy_rooms/The Greenholt Phish/Screenshots/06_DMARC-Record.png)

## Attachment Analysis
That's enough online tools, we will now analyze the attached file to the email. Firstly, we need to figure out the name of the file. I'll be using the search option again and look for either "attachment" or "filename" to discover any files that may be present

![Attachment](/Easy_rooms/The Greenholt Phish/Screenshots/07_filename.png)

For further analysis we are tasked to download the attachment to the lab machine. We don't want to actually open it, just download it, save it to the desktop and in your terminal, use sha256sum and the name of the file to calculate the Hash

![The Hash](/Easy_rooms/The Greenholt Phish/Screenshots/08_filehash.png)

Last but not least, an investigation on the calculated hash using [VirusTotal](https://www.virustotal.com/gui/home/search). Everything we need for the last two questions are found on the Details page when you enter the hash of the file there

![Virus Total Investigation](/Easy_rooms/The Greenholt Phish/Screenshots/09_file-details.png)

## Conclusion
A very easy but fun room! To sum things up, this is definitely a phishing Email if you couldn't tell. I might do another phishing Analysis challenge, look out for it!  
