# DC: 3
[VulnHub link](https://www.vulnhub.com/entry/dc-3,312/)  
By DCAU

* As with DC: 1 and DC: 2, we are first greeted with a login page that requires users to specify both the username and the password: 
![](/screenshots/dc-3/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-3/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.8`.
* The host is running only the HTTP (port 80) service, where the port for this server is open.
* Run `nmap -p- -A 10.0.2.8`:
![](/screenshots/dc-3/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see an Apache httpd web server running on port 80 (which is open). Also, it appears that the web server is running on a Joomla Content Management System (CMS) instead of WordPress.
* Opening `http://10.0.2.7` reveals a site with a welcome post (message) and a login form on the right-hand side:
![](/screenshots/dc-3/siteWebServer.jpg)
* Unlike the previous 2 iterations of DC which has 5 flags, this one only has 1 flag.
* I tried my luck on the login form with common login credentials such as `admin:admin` and `admin:password` which do not work as expected.
* Moreover, the displayed error message does not reveal to us if the username exists or not, unlike WordPress which does.
![](/screenshots/dc-3/loginInvalidMessage.jpg)
* The username is in red in the screenshot, but I do not think that means that the username does not exist. The username `admin` is likely to exist because we see that the welcome post was `Written by admin`, and I tried logins with the username `admin` and also other random ones, but both showed the username in red as a result of the invalid login attempt.
* Next, I tried `http://10.0.2.8/robots.txt`, where the output told us that a `robots.txt` file does not exist.
* The page source does not seem to reveal much of interest to us as well.
* It seems like there was nothing more I could poke at from the site by clicking around, so I searched for `joomla scanner kali` on Google, and found that our Kali VM has `joomscan` which scans for Joomla CMS vulnerabilities.
* Run `joomscan -u 10.0.2.8`:
![](/screenshots/dc-3/joomscan.jpg)
* The first thing that caught my eye was the `admin page`, which really actually is a login page for administrators. Again, I tried common login credentials which were not successful:
![](/screenshots/dc-3/siteAdminLogin.jpg)
* And once again, there is no information leakage on whether we have a working username or not.
* Next, I tried to access the 4 directory listings that were shown and we are actually able to see the various directories and files within them!