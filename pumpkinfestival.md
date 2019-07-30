# Mission-Pumpkin v1.0: PumpkinFestival
[VulnHub link](https://www.vulnhub.com/entry/mission-pumpkin-v10-pumpkinfestival,329/)  
By Jayanth

* Similar to the earlier challenges in this series, we are first greeted with a login page that requires users to specify both the username and the password. We are also given the IP address of the vulnerable machine: `10.0.2.15`.
![](/screenshots/pumpkinfestival/loginInitial.jpg)
* Because we are given the machine's IP address, we will skip the usual nmap scan of finding live hosts in our network and head straight for a more detailed scan with `nmap -p- -A 10.0.2.15`:
![](/screenshots/pumpkinfestival/hostFullScan.jpg)
* The vulnerable machine is running 3 services:
1. FTP - vsftpd 2.0.8 or later on Port 21 (the scan indicated 3.0.2 under FTP server status)
2. HTTP - Apache httpd 2.4.7 on Port 80
3. SSH - OpenSSH 6.6.1p1 Ubuntu on Port 6880
* With the exception of SSH on port 6880, the other 2 services are running on the port numbers that we expected.
* Let us first start off with exploring FTP on port 21, since we see from the nmap scan that Anonymous login is allowed.

# FTP on Port 21
* First, run `ftp 10.0.2.15` and then enter `anonymous` as the username and press `Enter` when prompted for the password - and we are in! We see the directory `secret` as expected from the nmap scan. Entering the directory, we see a file `token.txt`.
![](/screenshots/pumpkinfestival/ftpLogin.jpg)
* Run `get token.txt` to download the file, and we see a string `PumpkinToken : 2d6dbbae84d724409606eddd9dd71265`. Putting it through `hash-identifier` tells us that it is either a MD5 hash or "Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))". Putting them into online hash crackers resulted in no-match.
![](/screenshots/pumpkinfestival/tokenTxt.jpg)
* There are no other files so I guess that is all for now in this section! We will next move onto the HTTP service at port 80.

# HTTP on Port 80
* Running `10.0.2.15` in our Firefox greets us with 2 potential usernames - `jack` and `harry`, and the hint that PumpkinTokens can help us to "get to our pumpkins": 
![](/screenshots/pumpkinfestival/siteHTTPPage.jpg)
* Opening up the Page Source, we see a comment that tells `harry` to "find the pumpkin", and another PumpkinToken that is hidden in the background of the page: `PumpkinToken : 45d9ee7239bc6b0bb21d3f8e1c5faa52`.
![](/screenshots/pumpkinfestival/siteHTTPPagePumpkinToken.jpg)
* We also see that there is a script found near the top of the Page Source that prevents a right-click on the page.
* Opening up `robots.txt`, we see a short list:
![](/screenshots/pumpkinfestival/robotsTxt.jpg)
* I get a `404` on WordPress and `Forbidden` on tokens and users. Opening up `/store/track.txt`, we get a hit which gives jack's tracking code as `2542 8231 6783 486`:
![](/screenshots/pumpkinfestival/storeTrackTxt.jpg)
* `/store` and `/img` are also `Forbidden`. Trying `/tokens/[token_string]`, `/wordpress/wp-login.php` and `/users/[user_name]` (e.g. jack / harry / admin) did not work too.
* Running a `nikto` scan did not reveal anything additional to us as well: 
![](/screenshots/pumpkinfestival/niktoScan.jpg)
* `gobuster` did not turn up anything else of note either:
![](/screenshots/pumpkinfestival/gobusterScan.jpg)
* Got a little stuck here, so let us try out SSH on port 6880...

# SSH on Port 6880
* Trying a login using `ssh 10.0.2.15 -p 6880`, it seems like we cannot login using a password - it must be a private key. I guess brute-forcing our way in is not an option in this case.
![](/screenshots/pumpkinfestival/sshAttemptLogin.jpg)
* To-be-continued...