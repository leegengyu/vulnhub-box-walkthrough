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
* Running `hydra -l administrator -P /usr/share/wordlists/rockyou.txt 10.0.2.15 http-form-post '/index.php:uname=^USER^&psw=^PASS^&btnLogin=Login:S=Logout'` for 15 minutes gives us no results. I reran the command with the failure string being set to `psw`, but after 20 minutes, it yielded no results as well. Also tried another 2 runs with the username `admin` instead (with the 2 different success/failure strings).
* Running `nikto -h 10.0.2.15` tells us that the PHP version used is `4.3.9`. Also, it says that the `OPTIONS` HTTP method is enabled, but I could not get it to run. Nothing else much of interest.
![](/screenshots/kioptrix-level-2/niktoScan.jpg)
* Tried the `Apache mod_isapi Dangling Pointer` module on Metasploit ([corresponding Rapid7 page](https://www.rapid7.com/db/modules/auxiliary/dos/http/apache_mod_isapi)), with the RPORT being set to either 80 or 443, but to no avail.
* I tried a number of SQL injection attempts on the 2 login fields, but they did not work too: `' OR '1=1'`, `' OR 1=1`, `' OR '1=1' --`, `' OR '1=1 --'`.

## SSH service at Port 22 ##
* Tried to login as `root` - nothing special here (no banners or whatsover).
![](/screenshots/kioptrix-level-2/sshLoginAttemptRoot.jpg)

## Apache HTTPS service at Port 80 ##
* We add the site as a security exception as usual, and find the same site as was discovered in Port 80.

## MySQL service at Port 3306 ##
* Running `mysql -h 10.0.2.15` gives us an error message: `ERROR 1130 (HY000): Host '10.0.2.6' is not allowed to connect to this MySQL server`.

## CUPS service at Port 631 ##
* Accessing `10.0.2.15:631` reveals a `Forbidden` page. Intercepting its GET request reveals nothing of interest apart from what we have already picked up in our `nmap` scan:
![](/screenshots/kioptrix-level-2/cupsGetRequest.jpg)
* The same scan mentioned about HTTP methods, and so I changed the GET request to an OPTIONS one:
![](/screenshots/kioptrix-level-2/cupsOptionsRequest.jpg)
* Trying out both POST and PUT methods yielded the same `Forbidden` result.
* According to the [online documentation](https://help.ubuntu.com/lts/serverguide/cups.html), the CUPS service "can be configured and monitored using a web interface, which by default is available at http://localhost:631/admin". Trying to access the `/admin` page does not work out unfortunately.
* Based on the version `1.1` that we are aware of, there appears to be several exploits available for it, according to Exploit-DB.

# Concluding Remarks
To-be-added

# Other Walkthrough References
1. To-be-added