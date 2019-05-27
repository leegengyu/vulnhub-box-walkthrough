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
* Opening `http://10.0.2.8` reveals a site with a welcome post (message) and a login form on the right-hand side:
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
* The first thing that caught my eye was the `admin page` at `http://10.0.2.8/administrator/`, which really actually is a login page for administrators. Again, I tried common login credentials which were not successful:
![](/screenshots/dc-3/siteAdminLogin.jpg)
* And once again, there is no information leakage on whether we have a working username or not.
* Next, I tried to access the 4 directory listings that were shown and we are actually able to see the various directories and files within them!
* This was my first encounter with the Joomla CMS and I was not quite sure what to look out for, even if I had access to directories and files, unlike for example WordPress - where I knew that I would hunt for files like wp-config.php.
* However, we do know from `joomscan` that the Joomla CMS on this site was version `3.7.0`. I googled to see if there were exploits for this version of Joomla, and found the [Joomla Component Fields SQLi Remote Code Execution](https://www.rapid7.com/db/modules/exploit/unix/webapp/joomla_comfields_sqli_rce).
* I opened `msfconsole` and ran `use exploit/unix/webapp/joomla_comfields_sqli_rce`. Opening up the options, I set the RHOST to be `10.0.2.8` (**to be edited**):
![](/screenshots/dc-3/msfconsoleJoomlaOptions.jpg)
* However, upon running `exploit`, we see that no session was created because there was "no logged-in Administrator or Super User user found":
![](/screenshots/dc-3/msfconsoleJoomlaExploit.jpg)
* I am guessing that there are no users logged in at the moment, or at least there are no users which have administrative privileges who are logged in.
* I googled further and found a proof-of-concept exploit for the same vulnerability at this [https://github.com/XiphosResearch/exploits/tree/master/Joomblah](GitHub link).
* Clone the Git repository, and run `python joomblah.py http://10.0.2.8` (**to be edited**):
![](/screenshots/dc-3/joomblahOutput.jpg)
* Within a few seconds, we get to know that the user `admin` exists, along with his password hash `$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu`.
* I ran `hash-identifer` against the password hash and interestingly found no results:
![](/screenshots/dc-3/hashIdentifierAdmin.jpg)
* I extracted the front portions of the hash, i.e. `$2y$` and ran a Google search against it. Wikipedia tells us that such a prefix in a hash string indicates that the hash is a "bcrypt hash in modular crypt format".
* We will be using `hashcat`, ... **Find out if background of hashcat is required.**
* I ran `hashcat --help` to find out what parameters we had to use, and the `Hash modes` list came up to be pretty long, so I ran `hashcat --help | grep bcrypt` to find out what number was the bcrypt hash mode, which turned out to be `3200`.
* Insert the password hash into a text file (mine is `crackThisHash.txt`).
* Run `hashcat -m 3200 crackThisHash.txt /usr/share/wordlists/rockyou.txt --force`:
![](/screenshots/dc-3/hashcat.jpg)
* After a few seconds, we see that the hash has been cracked, and we have now got a valid set of credentials: `admin:snoopy`.
* Note: I used the standard given `rockyou.txt` wordlist for this purpose.
* Note: Running the command without `--force` for me resulted in this error:
![](/screenshots/dc-3/hashcatError.jpg)
* We are now able to log in to `http://10.0.2.8/administrator/`:
![](/screenshots/dc-3/loginAdmin.jpg)
* To-be-continued...