# Stapler: 1
[VulnHub link](https://www.vulnhub.com/entry/stapler-1,150/)  
By g0tmi1k

* We are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/stapler/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work. Also, not exactly sure why the initial login prompt and the one after an attempted login is different.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/stapler/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.18`.
* The host is running 7 services, where all of the listed ports below are open:
1. FTP at port 21
2. SSH at port 22
3. Domain at port 53
4. HTTP at port 80
5. NetBIOS-SSN at port 139
6. Doom at port 666
7. MySQL at port 3306
* There is also a closed port 20 that was running FTP-data service.
* This is our first walkthrough with so many services and ports open, probably because that there are multiple ways to get a limited shell and root access as mentioned by the author.
* Run `nmap -p- -A 10.0.2.18`:
![](/screenshots/stapler/hostFullScan.jpg)
* The scan results turned out to be really long because of the many services running.

# FTP at Port 21
* Our `nmap` scan results showed that anonymous FTP login is allowed, and from this [article](https://shahmeeramir.com/penetration-testing-of-an-ftp-server-19afe538be4b), I found out that we can search for anonymous login permissions using this metasploit exploit:
1. `msfconsole`
2. `use auxiliary/scanner/ftp/anonymous`
3. `set rhosts 10.0.2.18`
4. `exploit`
![](/screenshots/stapler/ftpAnonymous.jpg)
* While we managed to find out that we have `READ` permissions, we did not manage to find out the vsFTPd (Very Secure File Transfer Protocol Daemon) version. The information was deliberately replaced with a custom message. Our `nmap` scan only told us that the version is `2.0.8` or later.
* Nonetheless, the FTP server status section under the scan did state `vsFTPd 3.0.3 - secure, fast, stable`.

# Web Server at Port 80 and 12380
* Visiting `http://10.0.2.18` results in a Not Found error. It appears to be a PHP client server running at port 80:
![](/screenshots/stapler/siteWebServer.jpg)
* We had also found another web server running at port 12380 during our second, in-depth `nmap` scan. An Apache web server of version `2.4.18` is running at this port.
* Visiting `http://10.0.2.18:12380` results in proper site loading:
![](/screenshots/stapler/siteWebServerReal.jpg)
* It seems like there is only the default page available, because trying to load a random, non-existent section of the page resulted in the same page displayed. Same result when trying to load a file like `robots.txt`.
* The website is built by `Creative Tim`.
* Running `whatweb` on port 80 did not yield any useful results, but we find the status code of `400 Bad Request` and an extra HTTP header as part of the response for the latter port:
![](/screenshots/stapler/whatweb12380.jpg)
* According to what I found from Google, a Bad Request is due to our invalid request that the server is unable to process.
* `use auxiliary/scanner/http/apache_optionsbleed` on `msfconsole` (where this version of Apache HTTP server is affected by) did not work here either.

# Doom at Port 666
* Visiting `http://10.0.2.18:666/` (running service Doom, which I have no idea what it truly is) results in a page displaying unreadable information:
![](/screenshots/stapler/portDoom.jpg)

# MySQL at Port 3306
* Running on version `5.7.12-0ubuntu1`, I found an [exploit](https://www.exploit-db.com/exploits/40679) that would work on it, provided that we could get credentials to a system account.
* This [article](https://robert.penz.name/1416/how-to-brute-force-a-mysql-db/) shows us how we can do a dictionary attack using `hydra`.

# Concluding Remarks
Encountering a vulnerable machine with so many services makes things more challenging in my opinion because there appears to be so many attack vectors that we can target.

1. To-be-added

# Other walkthroughs visited
1. To-be-added