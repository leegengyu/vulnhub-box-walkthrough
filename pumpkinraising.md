# Mission-Pumpkin v1.0: PumpkinRaising
[VulnHub link](https://www.vulnhub.com/entry/mission-pumpkin-v10-pumpkinraising,324/)  
By Jayanth

* Similar to the earlier challenge in this series (PumpkinGarden), we are first greeted with a login page that requires users to specify both the username and the password. We are also given the IP address of the vulnerable machine: `10.0.2.5`.
![](/screenshots/pumpkinraising/loginInitial.jpg)
* Because we are given the machine's IP address, we will skip the usual nmap scan of finding live hosts in our network and head straight for a more detailed scan with `nmap -p- -A 10.0.2.5`:
![](/screenshots/pumpkinraising/hostFullScan.jpg)
* The vulnerable machine is running only 2 services:
1. SSH - OpenSSH 6.6.1p1 Ubuntu on Port 22
2. HTTP - Apache httpd (no identified version) on Port 80
* Both of them are running on the port numbers that we expected.
* Let us first start off with exploring SSH on port 22.

# SSH on Port 22
* Running `ssh 10.0.2.5`, we are prompted with `root`'s password as expected. There is no custom banner for us to grab - we will stop here for now for SSH since we do not have more clues and head straight for port 80.

# HTTP on Port 80
* Running `10.0.2.5` on our Firefox reveals this landing page. Hovering our cursor over each of the 4 images reveals a name and a description of the pumpkin.
![](/screenshots/pumpkinraising/siteLandingPage.jpg)
* In addition, we see the name `jack` once again - a username which popped up in the previous challenge, but we did not utilise him to get `root`.
* I tried to access the `/images` directory as was done in the previous challenge, but here no permissions were given to us.
* Opening up the Page Source shows us an interesting string in the comments: `VGhpcyBpcyBqdXN0IHRvIHJlbWFpbmQgeW91IHRoYXQgaXQncyBMZXZlbCAyIG9mIE1pc3Npb24tUHVtcGtpbiEgOyk=`.
![](/screenshots/pumpkinraising/siteLandingPageSource.jpg)
* Attempting to base-64 decode it, we get the message: `This is just to remaind you that it's Level 2 of Mission-Pumpkin! ;)`. Moving on...
* Opening up `robots.txt`, we see a whole page of information - even the comments at the top were left intact:
![](/screenshots/pumpkinraising/robotsTxt.jpg)
* The first file that caught my attention is `/hidden/note.txt`. Opening it up shows us what appears to be 3 sets of credentials:
1. Robert : C@43r0VqG2=
2. Mark : Qn@F5zMg4T
3. goblin : 79675-06172-65206-17765
![](/screenshots/pumpkinraising/hiddenNote.jpg)
* I thought robert and mark's passwords are base64-encoded, but they could not be decoded (I followed up and tried to base32-decode it after finding out that something after that was base32-encoded, but this did not work out either). I tried to use these 3 sets of credentials to log in via SSH but they did not work out. Not sure what these mean at this point in time.
* I visited `underconstruction.html` next, and I had just thought that there was nothing more to the page, but a pumpkin image with `+++ PAGE UNDER CONSTRUCTION +++`.
* However, it turns out that hovering our mouse over the image gave us the message `Looking for seeds? I ate them all!`, and selecting all elements told us a hidden message at the bottom of the page `jackolantern dot GraphicsInterchangeFormat is under images`:
![](/screenshots/pumpkinraising/constructionPage.jpg)
* I did not realise that there was more to the page until I examined the Page Source:
![](/screenshots/pumpkinraising/constructionPageSource.jpg)
* When I went to `/images/jackolantern.gif`, I found that it was just a static image, as compared to the moving `/images/uc.gif`. Hmm. Maybe there would be something more to `jackolantern.gif`, so I downloaded it and ran `strings` against it, but nothing noteworthy came out of it, though I did see `GIF89a` at the very beginning which reminded me of an earlier challenge that we did.
* Running `strings` against `uc.gif` did not result in anything substantial either.
* Opening `/seeds/seed.txt.gpg`, we see a bunch of garbled text, where I am not sure how this file can be used. I downloaded it and renamed it to `jpg` and `gif`, but it did not work out. I tried to find `/seeds/seed2.txt.gpg` and so on, but that did not work out either.
![](/screenshots/pumpkinraising/gpgFile.jpg)
* **Continuation**: After being stuck below, I came back to work on this, and searched about the file extension `.txt.gpg`. Turns out that this means that the original text key had been encrypted with a key using GPG. Running `gpg --decrypt seed.txt.gpg` prompts us for a key, and it also tells us that it contains AES256-encrypted data. I attempted with the 3 sets of login credentials found earlier, but to no avail.
* After trying to access several other directories and files from `robots.txt`, some of them were `Not Found` - perhaps it is not the most updated list.
* Running a `nikto` scan did not turn up information that we did not already know, though it did rightfullly highlight the 3 files which were the most useful findings:
![](/screenshots/pumpkinraising/niktoScan.jpg)
* `gobuster` did not turn up anything else of note either:
![](/screenshots/pumpkinraising/gobusterScan.jpg)
* At this point I was a little stuck, so I revisited the landing page and found that we missed out `pumpkin.html`, which was found by clicking on Pumpkin Seeds or by going again into the Page Source:
![](/screenshots/pumpkinraising/pumpkinHTML.jpg)
* Opening up the Page Source, we see another string in the comments: `F5ZWG4TJOB2HGL3TOB4S44DDMFYA====`. Not sure what it is, since I could not base64-decode it, nor could `hash-identifier`.
![](/screenshots/pumpkinraising/pumpkinHTMLSource1.jpg)
* At the end of the Page Source, we see another chunk of comments - what appears to be in hexadecimal.
![](/screenshots/pumpkinraising/pumpkinHTMLSource2.jpg)
* Using an online hexadecimal to ASCII converter, we see this message containing our first seed ID `96454`:
![](/screenshots/pumpkinraising/pumpkinHTMLHexToASCII.jpg)
* At this point, I was pretty much stuck, so I reached out for a lifeline and found that the earlier string which we could not decode (`F5ZWG4TJOB2HGL3TOB4S44DDMFYA====`) - was a **base32-encoded string**. Decoding it (online) shows us that it actually is `/scripts/spy.pcap`.
![](/screenshots/pumpkinraising/base32Decode.jpg)
* Downloading `spy.pcap`, we find that there are 42 packets within, all of which belong to the TCP protocol.
* The first 3 packets belong to the TCP handshake, and from the 4th packet onwards, we start seeing recognisable ASCII characters (i.e. messages) from the packets captured. Following that TCP stream, we see that we have the second seed ID `50609` (although we do not know which category of pumpkin seeds that belongs to):
![](/screenshots/pumpkinraising/wiresharkSeedId.jpg)
* Moving onto the next TCP stream, we see that there is another set of dialogue that took place. I suppose that if Robert still had `Jack-Be-Little` pumpkin seeds in stock, we would have got the seed ID for it as well:
![](/screenshots/pumpkinraising/wiresharkTCPStream1.jpg)
* And that was it - only 2 TCP streams within those packets captured.
* Thinking back, it was mentioned that jack was unaware that people can secretly spy online conversations - turns out that it was referring to the Wireshark capture packets that we saw!
* At this point, I got stuck again, and reached out for another lifeline - turns out that there was an encoded message within the `jackolantern.gif`! I had never experienced steganography being used in our Vulnhub challenges so far, and only got it once when I was playing on PicoCTF. On hindsight, the hint was quite apt - that the pumpkin had eaten the seeds meant that information about the seeds would be within the image (i.e. steganography)! Oh my.
* When I reached out to view a walkthrough on this part, I tried to use the online decoder that I had used during PicoCTF previously, but to no avail. It returned the same result as I would see when using `strings`. At a loss, I read on further about Steghide and StegoSuite, which were tools in Kali for steganography.
* Steghide did not work out, and turns out that we had to use StegoSuite. Moreover, it was `mark`'s password which was required to extract the information finally. Opening `decorative.txt`, we find the next seed ID: `86568`.
![](/screenshots/pumpkinraising/stegoSuiteSuccess.jpg)
* For the last seed ID, I knew that it had to do with `seed.txt.gpg`, because that was the only clue that we had not cracked. However, I had tried all means to get the password, but to no avail. I reached out for my third lifeline - the password is `SEEDWATERSUNLIGHT` (case-sensitive). Hmmmm - I would never have figured that out...
![](/screenshots/pumpkinraising/gpgFileCracked.jpg)
* The image that we see is actually morse code - we see `morse` being mentioned earlier as one of the seeds sellers. I tried the first two online decoders, which did not give very correct results. Though I could have pieced two and two together to get the last seed ID `69507`, I went to the walkthrough and found that the person used CyberChef, which gave the cleanest and most complete version (the one pictured at the bottom).
![](/screenshots/pumpkinraising/morseCodeDecode.jpg)
* Okay, we have got all 4 seeds. Now what? The only information that we have yet to fully utilise is 2 out of 3 of the credentials, and the fact that we have SSH.
* So there were several obstacles here. After identifying that the user `jack` is the most probable one (since it was mentioned that he is the only expert in raising healthy pumpkins), I thought that the password format would be in the form of `goblin`'s password, with the dashes. It turns out that the actual password did not have any dash, and was just a string of the seed IDs that we had found. There would have been 4! == 24 permutations, and only one is correct.
* **Post-mortem**: I learnt from one of the walkthroughs that we can enumerate SSH usernames using a metasploit module. Run `msfconsole`, then `auxiliary/scanner/ssh/ssh_enumusers`, then `set USER_FILE /usr/share/wordlists/seclists/Usernames/Names/names.txt` and finally `run`. The username list is taken from [SecLists](https://github.com/danielmiessler/SecLists), and that list contains 10,000+ names.
* **Note**: The SSH username enumeration is only possible if public key authentication is enabled on the SSH service.
* The credential that would get us in is `jack:69507506099645486568`:
![](/screenshots/pumpkinraising/sshLogin.jpg)
* Turns out that we are stuck in a `rbash` even though we are in. To get out of it, run `python -c 'import pty;pty.spawn("/bin/bash")'`.
* Navigating to one level above jack's home directory, we see that only his exists. `/etc/passwd` also concurs that he is the only user besides `root` that we would look at.
* We cannot navigate to `/root`, and running `sudo su` with jack's password tells us that `Sorry, user jack is not allowed to execute '/bin/su' as root on Pumpkin.`.
* Running `sudo -l` tells us that jack can run `/usr/bin/strace` as `root`.
* From here, it was easy for me to get the flag. Simply run `sudo strace /bin/sh` to spawn a shell as `root`, then navigate to `/root` and open up `flag.txt`. Hurray, we are done!
* Note: Using `/bin/bash` instead is possible I suppose, but it was pretty annoying not to be able to see my command in one line as I type when the read system call was waiting for the user input. I saw whatever I type fly upwards in the terminal with bin-bash.
* Alternative: Run `sudo strace -o /dev/null /bin/sh` so we do not see the system calls that `strace` would print. What this command additionally does is to redirect all its output to `/dev/null` (where the null device is a special device that discards the information being written into it) so we do not see it. With this command, `/bin/sh` or `/bin/bash` is fine.
![](/screenshots/pumpkinraising/flag.jpg)

# Concluding Remarks
I had thought that the second part of the challenge series would be manageable for me, i.e. without having to refer to any walkthroughs at all - but I did eventually, and had to do so more than once because I had exhausted attempts with what I had already known. This also brought me back to my intent of my continued practice to work on beginner-level CTFs, as I believe that there would still be gaps that I had to fill, and things to learn.

I felt that out of the several occasions where I got stuck, one or two were owing to my ownself where I did not push hard enough to piece together the clues. Firstly, the part about the base32-decoding. I did not know that it existed beforehand, but perhaps searching a little longer about other encoding styles might have revealed it. Secondly, the part about steganography. I stopped at Steghide and did not push forward with the other tool, thinking that the 3 credentials found could not have been used here (I had thought they were user credentials).

The part about the password being `SEEDWATERSUNLIGHT`, and `jack`'s password being the multiple seed IDs pieced together - those were truly unique and interesting situations. For the former, it would be very unlikely that I would have guessed it, I suppose. But for the latter, hitting a few more hours would have probably gotten me the password.

1. Learnt that there were other base type encodings as well, specifically the base32 encoding in this challenge. After I found out that it was base32-encoded, I asked myself - how would one know? Moreover, I had always been experiencing only base64-encoded content so far. I found 2 stackexchange questions - [here](https://security.stackexchange.com/questions/186815/identify-encoding-type-decoding-base-32-64) and [here](https://security.stackexchange.com/questions/3989/how-to-determine-what-type-of-encoding-encryption-has-been-used) that were useful. To sum it up, use experience to make educated guesses, and also some signs such as `==` to tell that it was base64-encoded, etc.
2. Learnt to really think better about the challenge's hints that were dropped - `secretly spy online conversations`, `Looking for seeds? I ate them all!`, etc.
3. Learnt about steganography tools in Kali, specifically Steghide and StegoSuite
4. Learnt about privilege escalation with `strace`. I knew how this tool worked since my school work required me to go through system calls with it, but using it for the first time to escalate ourselves was interesting.

All in all, this was definitely a good one for me!

# Reference Links
1. https://www.hackingarticles.in/pumpkinraising-vulnhub-walkthrough/
2. https://ethicalhackingguru.com/mission-pumpkin-level-2-pumpkinraising-vulnhub-walkthrough/