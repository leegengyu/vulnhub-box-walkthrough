# Kioptrix: Level 1.1 (#2)
[VulnHub link](https://www.vulnhub.com/entry/kioptrix-level-11-2,23/)  
By Kioptrix

## Setting Up the Box ##
* The updated version, i.e. second release of the box is the one used here (and not the original release).
* The solution provided in kioptrix-level-1 to set up the box works for this one as well.

## Enumeration ##
* We are first greeted with a login page that requires users to specify a set of credentials:
![](/screenshots/kioptrix-level-2/loginInitial.jpg)
* Run `nmap 10.0.2.*`, where we find `10.0.2.15` to be the IP address of the vulnerable machine. 6 services were also discovered, where they are all in an Open state.
![](/screenshots/kioptrix-level-2/nmapScan.jpg)
* Run `nmap -sC -sV 10.0.2.15` to enumerate the running services:
![](/screenshots/kioptrix-level-2/hostFullScan.jpg)
* Let us start with the Apache httpd service at port 80.

## Apache httpd service at Port 80 ##
* We see a `Remote System Administration Login` page after loading `http://10.0.2.15/` on our browser:
![](/screenshots/kioptrix-level-2/httpHomePage.jpg)
* We note an interesting comment in the page source that probably gives us an existing username `administrator`:
![](/screenshots/kioptrix-level-2/httpHomePageSource.jpg)
* An invalid login attempt does not reveal any error messages.
* No `robots.txt` file was found.
* Running `gobuster dir -u http://10.0.2.15 -w /usr/share/wordlists/dirb/common.txt` does not yield anything of much interest.  (Running the same command and adding the search for .php pages did not yield new results.)
![](/screenshots/kioptrix-level-2/httpGobusterDirScan.jpg)
* Accessing `/manual` leads us to a default page with the `Apache HTTP Server Version 2.0 Documentation`.
* Back to the home page with the login screen, examining a login attempt as a POST request in Burp Suite sets us up for a `hydra` brute-force attempt:
![](/screenshots/kioptrix-level-2/httpHomePageLoginAttempt.jpg)
* Running `hydra -l administrator -P /usr/share/wordlists/rockyou.txt 10.0.2.15 http-form-post '/index.php:uname=^USER^&psw=^PASS^&btnLogin=Login:S=Logout'` for 15 minutes gives us no results. I am re-running the command with the failure string being set to `psw`: to-be-updated...
* Running `nikto -h 10.0.2.15` tells us that the PHP version used is `4.3.9`. Nothing else much of interest.
![](/screenshots/kioptrix-level-2/niktoScan.jpg)
* Tried the `Apache mod_isapi Dangling Pointer` module on Metasploit ([corresponding Rapid7 page](https://www.rapid7.com/db/modules/auxiliary/dos/http/apache_mod_isapi)), with the RPORT being set to either 80 or 443, but to no avail.

## SSH service at Port 22 ##
* Tried to login as `root` - nothing special here (no banners or whatsover).
![](/screenshots/kioptrix-level-2/sshLoginAttemptRoot.jpg)

## Apache HTTPS service at Port 80 ##
* We add the site as a security exception as usual, and find the same site as was discovered in Port 80.

## MySQL service at Port 3306 ##
* Running `mysql -h 10.0.2.15` gives us an error message: `ERROR 1130 (HY000): Host '10.0.2.6' is not allowed to connect to this MySQL server`.

# Concluding Remarks
To-be-added

# Other Walkthrough References
1. To-be-added