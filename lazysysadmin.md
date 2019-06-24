# LazySysAdmin: 1
[VulnHub link](https://www.vulnhub.com/entry/lazysysadmin-1,205/)  
By Togie Mcdogie

* We are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/lazysysadmin/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/lazysysadmin/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.14`.
* The host is running 6 services, notably SSH at port 22 and HTTP at port 80. All of the ports running a service are found to be open.
* Run `nmap -p- -A 10.0.2.14`:
![](/screenshots/lazysysadmin/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, for a start we can see OpenSSH running at port 22 and an Apache web server running on port 80.
* Visiting `http://10.0.2.14` shows us a non-standard site, i.e. it is not simply taken entirely from a template:
![](/screenshots/lazysysadmin/siteWebServer.jpg)
* It appears that the site is powered by Silex.
* Opening up the `robots.txt` file shows us:
![](/screenshots/lazysysadmin/robotsTxt.jpg)
* `/old` and `/test` are empty directories, while `/TR2` is not found. The last directory, `/Backnode_files` is the only directory that is useful. It contains the files (e.g. images) that we see on the front page)
![](/screenshots/lazysysadmin/backnodeFilesDirectory.jpg)
* I ran `gobuster -e -u http://10.0.2.14 -w /usr/share/wordlists/dirb/common.txt` to find out if there were other files and directories that we could access:
![](/screenshots/lazysysadmin/gobusterResults.jpg)
* There are 2 results that stood out: `/phpmyadmin` and `/wordpress`.
* `/phpmyadmin` page shows a login page:
![](/screenshots/lazysysadmin/phpMyAdminPage.jpg)
* Attempting a login results in an error, stating that we are unable to log in to the MySQL server because the connection failed. I guess there is no chance for us to do a wordlist attack here for now then.
![](/screenshots/lazysysadmin/phpMyAdminPageAttemptedLogin.jpg)
* `/wordpress` brings us to another site:
![](/screenshots/lazysysadmin/wordpressTR2.jpg)
* The post titled `Hello world!` is posted by user `admin`:
![](/screenshots/lazysysadmin/wordPressTR2Post.jpg)
* Since this is a WordPress site, let us try to visit `http://10.0.2.14/wordpress/wp-login.php` - which gives us our login page for this CMS! An invalid login attempt with username being `admin` confirms that the user exists.
* Running `wpscan -v --url http://10.0.2.14/wordpress` tells us that the WordPress version is `4.8.1`, and that there are 27 vulnerabilities detected:
![](/screenshots/lazysysadmin/wpscan.jpg)
* Side-track: Assuming we could log in as a low-level user, we could execute [WordPress 3.7-5.0 (except 4.9.9) - Authenticated Code Execution](https://wpvulndb.com/vulnerabilities/9222). However, we see in the next scan that we have only got the `admin` user account available.
* Running `wpscan --url http://10.0.2.14/wordpress --enumerate u` tells us that the scanner could only find one username `admin` for the CMS:
![](/screenshots/lazysysadmin/wpscanEnumUsers.jpg)
* Note: I tried `togie` as a username since there were many lines of it in the `Hello world!` post, but it does not exist.
* Detecting the plugins used reveals nothing of significance - `wpscan -v --plugins-detection aggressive --url http://10.0.2.14/wordpress`. There is only 1 plugin, `akismet` detected.
![](/screenshots/lazysysadmin/wpscanEnumPlugins.jpg)
* I ran `gobuster -e -u http://10.0.2.14/wordpress -w /usr/share/wordlists/dirb/common.txt` to see if there were unexplored files and directories but nothing interesting came up as well:
![](/screenshots/lazysysadmin/gobusterResultsWordPress.jpg)
* Note: `robots.txt` file could not be found.
* Run `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.14/wordpress http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Dashboard'` to attempt to get user `admin`'s password.
* To-be-continued

# Concluding Remarks
1. To-be-added

# Other walkthroughs visited
1. To-be-added