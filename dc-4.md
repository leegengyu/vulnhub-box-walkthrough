# DC: 4
[VulnHub link](???)  
By DCAU

* As with the first 3 iterations of the DC-series, we are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/dc-4/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-4/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.10`.
* The host is running 2 services, SSH at port 22 and HTTP at port 80. Both ports are found to be open.
* Run `nmap -p- -A 10.0.2.10`:
![](/screenshots/dc-4/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see an nginx web server running on port 80 (which is open). This is the first time that we have encountered a non-Apache web server - not that it makes any difference (I think?).
* Opening `http://10.0.2.10` reveals a site with only an "Admin Information Systems Login":
![](/screenshots/dc-4/siteWebServer.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work. Moreover, any invalid attempts does not result in an error message being displayed. Instead, the 2 user input text fields are simply emptied. The page is also changed to `http://10.0.2.10/index.php`.
* The page source does not reveal anything of interest, and the robots.txt page does not exist (404 Not Found).
* The page is running on php - perhaps it could be a WordPress site? I tried to load `http://10.0.2.10/wp-login.php`, but was greeted with the message "No input file specified.". Hmm, interesting. It seems to be expecting a file at the login page?
* Time to find out what other pages or directories that we can access from the site - I tried using `uniscan` but it only gave 2 results which renders the same login page that greeted us.
![](/screenshots/dc-4/uniscan.jpg)
* Using `dirbuster`, we did not find out much additional useful information - the only page with 200 OK is what we already know. The rest of the pages attempted lead to either a 403 Forbidden or 302 Redirect.
![](/screenshots/dc-4/dirbuster.jpg)