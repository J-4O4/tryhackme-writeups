
# Checkmate Write-up

![Picture of the room](/Easy_rooms/Checkmate/Screenshots/00_Room.png)

Marco Bianchi deployed several internal services, yet due to deadlines and operational pressure, he reused weak and predictable passwords. Our objective will be to do a password security assessment

## Level 1

Visiting the page **http(://)MACHINE_IP:5000** will provide us with information for each level. For this level, we get the following information

```txt
Marco deployed a firewall at firewall.thm:5001 but kept default credentials.
```
> We can visit the each page with the IP as well and just changing the port number. This way we wont need to change the name in our `/etc/hosts` file for every level

The service "FirewallOS" is hosted there

![Level 1 Firewall](/Easy_rooms/Checkmate/Screenshots/01_lvl1.png)

I first of all tried to look up default credentials for that firewall, and stumbled upon credential pairs like `admin:admin` or `admin:root`. Yet all of them didn't work as they weren't specific to FirewallOS. However, admin should be the default name, so why not just try to bruteforce the password. 
For this, I'll be using ffuf for this, as I had some problems with hydra

```bash
ffuf -w /usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-10000.txt -X POST -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.113.146.22:5001/login -fc 200
```

I am using a rather small wordlist here for now since it's only level 1 and we are in a CTF environment. 

![Level 1 Password](/Easy_rooms/Checkmate/Screenshots/02_lvl1pass.png)

Enter the password on the main site which is hosted on port 5000 to get information about level 2

## Level 2

```txt
Marco built an internal Employee Login panel on jobs.thm:5002 and used common company keywords as passwords.
```
> Again, the MACHINE_IP will be enough so no need to change the `/etc/hosts` file

We'll visit the page hosted on port 5002 and go to the login form located at `/login`

Potentially, we could use a big wordlist such as `rockyou.txt`, however, that would take too long and isn't really effective. with the information given, you should probably get the idea to build your own wordlist with words on the page. I used a tool called **CeWL** for this, it's a web crawler which will look for words or emails on that page and add it to a wordlist. It's pre installed on the attackbox, so run the following command:

```bash
cewl -d 2 -m 3 --lowercase --with-numbers -w cewl_words.txt http://TARGET_IP:5002
```

After that, run a bruteforce with ffuf again

```bash
ffuf -w cewl_words.txt -X POST -d "username=marco&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -u http://TARGET_IP:5002/login -fc 200
```
> I used marco as the default name, as it was given by the login form as the default/example

![Level 2 Password](/Easy_rooms/Checkmate/Screenshots/03_lvl2pass.png)

Again, enter the password at the main site, port 5000 to proceed 

## Level 3

```txt
Navigate to social.thm:5003 and derive Marco's password from personal info.
```

Navigating there, we'll see another hint, which tells us to use the page from the previous level to build a password. From the previous level, these are the information we have gotten about marco

![Marco personal information](/Easy_rooms/Checkmate/Screenshots/04_marcoinfo.png)

For the username, I'll be sticking with "marco", or we might even have to figure out a valid username. 

I first of all tried common patterns like the name and the birthday, or only the birthday etc. However, non of these combinations worked. I've been stuck for quite a while manually figuring out a password. I got tired of that and did some research to see if there would be an easier way to do such thing. I am not quite a fan of the google AI assistant, but this time it actually helped me out. It suggested a tool called **[CUPP](https://github.com/mebus/cupp)** hosted on GitHub. 

I cloned it to the machine and ran the following:

```txt
git clone https://github.com/Mebus/cupp.git
cd cupp
chmod +x cupp.py
python3 cupp.py -i
```

Then fill out the following

![Usage of cupp.py](/Easy_rooms/Checkmate/Screenshots/05_cupp-py.png)

Then leave the rest empty and let the tool generate custom passwords for that person. After it is done, we can try to bruteforce it again using ffuf

```bash
ffuf -w marco.txt -X POST -d "username=marco&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -u http://10.113.146.22:5003/login -fc 200
```

You'd love to see it, we got a hit

![Level 3 Password](/Easy_rooms/Checkmate/Screenshots/06_lvl3pass.png)

## Level 4

```txt
On social.thm:5003, Marco recently uploaded a new profile picture. For privacy and storage consistency, the platform automatically renames uploaded files to the SHA256 hash of the original filename and saves them in the format (SHA256).png. Your task is to identify the original filename of Marco’s uploaded profile picture. Submit only the filename to proceed.
```

Now this is rather a hash cracking exercise than a password cracking one. After logging in to the page from the previous question, you'll quickly notice there isn't anything intractable. In order to see the file name of the profile picture, right click it and select "Open image in New Tab"

![Profile picture](/Easy_rooms/Checkmate/Screenshots/07_pfp.png)

Copy the name of the profile picture (See URL) without the `.png` and try to crack it with a tool like hashcat. You could use a online tool for this like [hashes.com](https://hashes.com/en/decrypt/hash), however, usually it's better to use an offline tool like hashcat since every hash you upload will also be uploaded to a foreign server. In a CTF practice like this, that doesn't matter, but when dealing with sensitive data, you should crack them on your own machine. Until recently, I also used online tools to crack them (I even made a whole write-up with them for [Crack_the_hash](/Easy_rooms/Crack_the_hash/Crack%20the%20hash)), but now I prefer hashcat. Here is how

```bash
1. echo 'd34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b' >> hash.txt

2. hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```
> The hash type for SHA-256 in hashcat is 1400, thus the `-m 1400`

![Cracked hash](/Easy_rooms/Checkmate/Screenshots/08_lvl4hash.png)

## Level 5

```txt
Marco has revealed his password pattern on social.thm:5003, using predictable rules based on keywords and formatting. Use this information to generate a targeted wordlist and brute-force the SSH service with username marco.
```

The post they are talking about is this one right here: 

![Post from Marco](/Easy_rooms/Checkmate/Screenshots/09_Marcopost.png)

I first of all tried manuell brute forcing. I tried to ssh into the machine with the command

```bash
ssh marco@TARGET_IP
```

I first of all tried the word from Level 2 as the company word, capitalized the beginning, and added `2024!` to the end. That didn't work

It was then when I noticed there were some tags for this post, the second tag is a company word marco already used, so maybe he added another word he uses as a tag. I then tried to do the same with `security` and it worked! 

![SSH login](/Easy_rooms/Checkmate/Screenshots/10_lvl5.png)

I know, this room probably expected you to somehow make a custom wordlist with rules or masking and use hydra to get access. However what I just did works as well and definitely saves time. If that would have not worked, I would've tried what I just explained

## Conclusion

That was a great refresher on password cracking, as this room requires you to use a variety of techniques to crack passwords efficiently. Especially if you've been enrolled in the module the challenge is part of - such as me. I had to try different approaches for each level which made me come out of my comfort zone, instead of sticking to the same techniques over and over again. I hope this write-up helped you out! 
