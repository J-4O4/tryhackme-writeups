
# Crack the hash Write-up

![Picture of the room](/Easy_rooms/Crack_the_hash/Screenshots/00_Room.png)

Hashes I'd say are a pretty important part of Cyber security, especially in Cyber threat intel or in securing database entries. However, just because they are generated through a one-way function, doesn't mean they are completely secure - nothing in the digital world is 100% secure.

Hashes can easily be cracked with online databases and wordlists, or tools such as hashcat or jumbo john. In order for hashes to be crackable, they must be in a provided wordlist or database, where your input is being sent to a hash function and compared to every other hash.

Enough talking, let's start!

## Level 1

This is our first hash we have to crack:

```txt
48bb6e862e54f2a795ffc4e541caed4d
```

For this, I didn't even need a tool like [Hash type identifier](https://hashes.com/en/tools/hash_identifier), as it's pretty obvious that this is a MD5 hash! You can tell by the fact that every letter is lower case and it's only 32 characters long.

I found this cool [website](https://md5decrypt.net/en/) to decrypt MD5 hashes and other hashes. Paste the hash in there for the first answer

![Hash 1](/Easy_rooms/Crack_the_hash/Screenshots/01_hash1.png)

Next up, we have this hash right here:

```txt
CBFDAC6008F9CAB4083784CBD1874F76618D2A97
```

It's probably a SHA1 hash i'd guess, since it's only 40 characters, hashes that are more secure like SHA256 have more characters. Since I am unsure, we can use [Hash type identifier](https://hashes.com/en/tools/hash_identifier) 

This confirmed my thesis, so I proceeded to go to the [SHA-1 Decrypt](https://md5decrypt.net/en/Sha1/)  section of the previously mentioned page

![Hash 2](/Easy_rooms/Crack_the_hash/Screenshots/02_hash2.png)

Awesome. The following hash now is:

```txt
1C8BFE8F801D79745C4631D09FFF36C82AA37FC4CCE4FC946683D7B336B63032
```

This looks like a SHA-256 hash to me, judging by the length of the hash. Again, use the online Hash type identifier to confirm your thesis before randomly trying to crack the hash

In this case, I was right again, so let's crack that using the [SHA-256 Decrypt](https://md5decrypt.net/en/Sha256/) section of the website.

![Hash 3](/Easy_rooms/Crack_the_hash/Screenshots/03_hash3.png)

Increasing the difficulty, we get this hash

```txt
$2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom
```

I am guessing it's a `bcrypt $2y$` hash, judging by the fact it starts with `$2y$`. However, I am not exactly familiar with bcrypt hashes so I used the Hash identifier again

![Type of hash 4](/Easy_rooms/Crack_the_hash/Screenshots/04_type-h4.png)

Interestingly, two possible hash types are named by the tool, so I decided this time, I'll use their hashing tool to crack this hash (As there is a [Decrypt Hashes](https://hashes.com/en/decrypt/hash/) section) .

![hash 4](/Easy_rooms/Crack_the_hash/Screenshots/05_hash4.png)
> I didn't censor it, as the hash type identifier somehow already cracked the hash when listing the hash type

Last but not least for this level, we are presented this hash right here

```txt
279412f945939ba78ce0758d3fd83daa
```

At first I thought it was a MD5 hash again, however I looked it up and turns out it is a MD4 hash instead. I went back to the other website as it had a section for [MD4 Decrypt](https://md5decrypt.net/en/Md4/)

![hash 5](/Easy_rooms/Crack_the_hash/Screenshots/06_hash5.png)


## Level 2

Alright, time to spice things up, shall we?

This level introduces hashes with a salt. A salt is basically a way to make the output of a hash function more unique when providing the same input. It's especially awesome for managing passwords in a database, so an attacker would have to crack each hash one by one instead of cracking one and finding out that X amount of people have the same password

This is the first hash in Level 2:

```txt
 F09EDCB1FCEFC6DFB23DC3505A882655FF77375ED8AA2D1C13F640FCCC2D0C85
```
> I wont mention the fact I checked the hash with the Hash type identifier, as it should be clear by now

Again, this is a SHA-256 hash, time to crack it with the previously mentioned website [SHA-256 Decrypt](https://md5decrypt.net/en/Sha256/)

![hash 6](/Easy_rooms/Crack_the_hash/Screenshots/07_hash6.png)

Okay, that was still pretty easy. Let's see how the next one will be:

```txt
1DFECA0C002AE40B8619ECF94819CC1B
```

This is a NTLM hash, used by Microsoft to manage passwords in the SAM (Security Account Management). For this hash, I used this [hash decrypt](https://hashes.com/en/decrypt/hash/?r=7) website

![hash 7](/Easy_rooms/Crack_the_hash/Screenshots/08_hash7.png)

Pay attention now, this is a hash with the salt: `aReallyHardSalt`:

```txt
$6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02.
```

Please note, I am aware there are tools like hashcat to make this process more efficiently, as you can provide a list of hashes and a custom wordlist which these online tools can't. However, I still need to get familiar with hashcat, that's why I stick to online tools (for now...)

I am using the website I used for the previous hash, since it also allows to crack hashes with a slat

![hash 8](/Easy_rooms/Crack_the_hash/Screenshots/09_hash8.png)
> I really have to say, this website is a blessing, definitely recommend

Finally, the last hash for this whole challenge has the slat `tryhackme` and would be:

```txt
e5d8870e5bdd26602cab8dbe07a942c8669e56d6
```

The hash decrypt site names the following format for hashes with a slat: `hash:salt`

Let's enter that and see what the website gives us as a result

![hash 9](/Easy_rooms/Crack_the_hash/Screenshots/10_hash9.png)

## Conclusion

Again, a quick and easy challenge which in my opinion helps one to get comfortable with hashes. Rather you are a blue teamer or read teamer, this room is really useful for both. I already mentioned that tools such as hashcat are a thing and should be preferred over online tools, but for this room, online tools were enough. When I get familiar with hashcat, I might do a hashing room with it if I find one! Thank you so much for reading to the end, I hope this write-up helped you out! 