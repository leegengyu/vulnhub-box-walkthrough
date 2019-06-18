# DC: 4
[VulnHub link](https://www.vulnhub.com/entry/dc-4,313/)  
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
* I attempted a `wpscan` command against the web server but it did not detect the site's CMS as WordPress.
* Running `whatweb -v 10.0.2.10` also did not tell us what CMS the site was on, except information that we had already known before (e.g. nginx version):
![](/screenshots/dc-4/whatweb.jpg)
* Note: `whatweb` is a web scanner that identifies technologies used by websites.
* Time to find out what other pages or directories that we can access from the site - I tried using `uniscan` but it only gave 2 results which renders the same login page that greeted us.
![](/screenshots/dc-4/uniscan.jpg)
* Using `dirbuster`, we did not find out much additional useful information - the only page with 200 OK is what we already know. The rest of the pages attempted lead to either a 403 Forbidden or 302 Redirect.
![](/screenshots/dc-4/dirbuster.jpg)
* Without making much of a headway, I decided that we will use a dictionary attack against the login page, using `hydra`: `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.10 http-post-form '/login.php:username=^USER^&password=^PASS^:S=Logout'`:
![](/screenshots/dc-4/hydra.jpg)
* I had several issues with running the command here, though I had already used it in 2 walkthroughs previously. The first issue was that instead of `login.php`, I was using `index.php` which did not work. I am rather puzzled as to why this is the case because `dirbuster` told us that accessing the former would result in an indirect to the latter. Accessing the former itself would also result in the same.
* Note: We discovered earlier that accessing `10.0.2.10` itself did not result in a redirect to `http://10.0.2.10/index.php`. Only an invalid login attempt did so. When logging in with `admin:happy`, entering the set of credentials brought us again to `index.php` first, where we had to enter the set of credentials again. So, I decided to head straight to `index.php` and enter the set of credentials, but again I was forced to enter it a second time. Heading straight to `login.php` also required me to log in twice. The mystery behind what puzzled me is still not solved, I guess.
* Note2: Perhaps having to log in twice is a mechanism to defeat brute-force/dictionary attacks, but `login.php` somehow works.
* The second issue was that I did not know what to expect upon a successful log in, i.e. what texts there would be. I was trying out words such as Welcome, and instead of Logout, I was trying Log Out (with a whitespace) instead.
* Note that we also had to make a guess with the username `admin` because an invalid login did not reveal to us if the attempted username even existed.
* This is the page that greets us after logging in:
![](/screenshots/dc-4/adminLogin.jpg)
* There are not a lot of words in the page, so your word/phrase that detects for a successful login in your `hydra` command must be accurate.

# Other walkthroughs visited
1. https://s1gh.sh/vulnhub-dc-4/