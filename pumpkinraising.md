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
* Attempting to base-64 decode it, we get the message: This is just to remaind you that it's Level 2 of Mission-Pumpkin! ;). Moving on...
* Opening up `robots.txt`, we see a whole page of information - even the comments at the top were left intact:
![](/screenshots/pumpkinraising/robotsTxt.jpg)
* The first file that caught my attention is `/hidden/note.txt`. Opening it up shows us what appears to be 3 sets of credentials:
1. Robert : C@43r0VqG2=
2. Mark : Qn@F5zMg4T
3. goblin : 79675-06172-65206-17765
![](/screenshots/pumpkinraising/hiddenNote.jpg)
* I thought robert and mark's passwords are base64-encoded, but they could not be decoded so I guess that is their dirct password.
* I tried to use these 3 sets of credentials to log in via SSH but they did not work out.
* I visited `underconstruction.html` next, but it just showed us a pumpkin image with "+++ PAGE UNDER CONSTRUCTION +++". Nothing in the page source of note, except this title of the image: Looking for seeds? I ate them all!.
* Opening `/seeds/seed.txt.gpg`, we see a bunch of garbled text, which is probably a set of keys, I think. Looking back at the challenge's description, we are required to find 4 pumpkin seeds - this should be the first one. I tried to find `/seeds/seed2.txt.gpg` and so on, but that did not work out.
![](/screenshots/pumpkinraising/gpgFile.jpg)
* After trying to access several other directories and files from `robots.txt`, I found out that the list is probably not the most updated - some of them were Not Found.
* Running a `nikto` scan did not turn up information that we did not already know, though it did rightfullly highlight the 3 files which were the most useful findings:
![](/screenshots/pumpkinraising/niktoScan.jpg)
* At this point I was a little stuck, so I revisited the landing page and found that we missed out `pumpkin.html`, which was found by clicking on Pumpkin Seeds or by going again into the Page Source:
![](/screenshots/pumpkinraising/pumpkinHTML.jpg)
* Opening up the Page Source, we see another string in the comments: `F5ZWG4TJOB2HGL3TOB4S44DDMFYA====`. Not sure what it is, since I could not base64-decode it, nor could `hash-identifier`.
![](/screenshots/pumpkinraising/pumpkinHTMLSource1.jpg)
* At the end of the Page Source, we see another chunk of comments - what appears to be in hexadecimal.
![](/screenshots/pumpkinraising/pumpkinHTMLSource2.jpg)
* Using an online hexadecimal to ASCII converter, we see this message - which contains an ID `96454`:
![](/screenshots/pumpkinraising/pumpkinHTMLHexToASCII.jpg)
* To-be-continued...