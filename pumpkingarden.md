# Mission-Pumpkin v1.0: PumpkinGarden
[VulnHub link](https://www.vulnhub.com/entry/mission-pumpkin-v10-pumpkingarden,321/)  
By Jayanth

* We are first greeted with a login page that requires users to specify both the username and the password. This is the most user-friendly GUI-based login screen that I have encountered thus far. We see the date on the top-right-hand corner and some friendly introductory text. The best part is that we are given the IP address that the machine is running on - `10.0.2.16`!
![](/screenshots/pumpkingarden/loginInitial.jpg)
* Because we are given the machine's IP address, we will skip the usual nmap scan of finding live hosts in our network and head straight for a more detailed scan with `nmap -p- -A 10.0.2.16`:
![](/screenshots/pumpkingarden/hostFullScan.jpg)
* The vulnerable machine is running 3 services:
1. FTP - vsftpd 2.0.8 or later on Port 21
2. HTTP - Apache httpd 2.4.7 on Port 1515
3. SSH - OpenSSH 6.6.1p1 on Port 3535
* Both HTTP and SSH are both running on non-conventional ports.
* Let us first start off with exploring FTP on port 21, since we already see from the nmap scan that anonymous login is allowed, and 1 file had been found.

# FTP at Port 21
* First, run `ftp 10.0.2.16` and then enter `anonymous` as the username and press `Enter` when prompted for the password - and we are in! We see `note.txt` as expected from the nmap scan.
![](/screenshots/pumpkingarden/ftpLogin.jpg)
* Run `get note.txt` to download the file, and we see a message whose only useful bit is probably the name `jack`:
![](/screenshots/pumpkingarden/noteTxt.jpg)
* Heading back into our FTP session, we find that we are already at the root directory, and that there are no other files. I guess that is all for now here! We will next move onto the HTTP service at port 1515.

# HTTP at Port 1515
* Running `10.0.2.16:1515` in our Firefox greets us with a message about a route map:
![](/screenshots/pumpkingarden/siteHTTPPage.jpg)
* Opening up the Page Source, we see a comment that says that pumpkin images would be something that can help us find a way to the route map:
![](/screenshots/pumpkingarden/siteHTTPPageSource.jpg)
* I opened the image file that we see on the loading page, and found that its URL is `http://10.0.2.16:1515/img/pumpkin.gif`.
* Going to the `/img` directory, we find a couple of images, and a `hidden_secret` directory:
![](/screenshots/pumpkingarden/imgDirectory.jpg)
* Entering the secret directory, we find `clue.txt` which contains a string `c2NhcmVjcm93IDogNVFuQCR5`. I ran `hash-identifier` against this, but not no hash type was detected.
* I tried to base-64 decode this, and found that it gave us `scarecrow : 5Qn@$ya`. Seems like it is a set of login credentials. Tried to login back on FTP with them, but to no avail.
* I opened up the rest of the images and found that they were simply pumpkin images.
* There is no `robots.txt` or `readme.html` file or `/wp-login.php` page detected.
* Running a `nikto` scan did not reveal anything additional to us as well:
![](/screenshots/pumpkingarden/niktoScan.jpg)
* A `gobuster` scan was not of help either:
![](/screenshots/pumpkingarden/gobusterScan.jpg)
* Let us move onto the last service - SSH at port 3535.

# SSH at Port 3535
* I tried `scarecrow:5Qn@$ya` here, but it did not work out.