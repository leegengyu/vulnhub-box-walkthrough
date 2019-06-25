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

# Apache Web Server at Port 80
* Visiting `http://10.0.2.14` shows us a non-standard site, i.e. it is not simply taken entirely from a template:
![](/screenshots/lazysysadmin/siteWebServer.jpg)
* It appears that the site is powered by `Silex`.
* Maybe because the number of users of Silex is little, because `searchsploit silex` throws up nothing.
* Opening up the `robots.txt` file shows us:
![](/screenshots/lazysysadmin/robotsTxt.jpg)
* `/old` and `/test` are empty directories, while `/TR2` is not found. The last directory, `/Backnode_files` is the only directory that is useful. It contains the files (e.g. images) that we see on the front page)
![](/screenshots/lazysysadmin/backnodeFilesDirectory.jpg)
* I ran `gobuster -e -u http://10.0.2.14 -w /usr/share/wordlists/dirb/common.txt` to find out if there were other files and directories that we could access:
![](/screenshots/lazysysadmin/gobusterResults.jpg)
* There are 3 results that stood out: `info.php`, `/phpmyadmin` and `/wordpress`.
* `info.php` shows a whole page of PHP information - not sure if there is supposed to be anything noteworthy in here:
![](/screenshots/lazysysadmin/infoPHP.jpg)
* `/phpmyadmin` page shows a login page:
![](/screenshots/lazysysadmin/phpMyAdminPage.jpg)
* Attempting a login results in an error, stating that we are unable to log in to the MySQL server because the connection failed. I guess there is no chance for us to do a wordlist attack here for now then.
![](/screenshots/lazysysadmin/phpMyAdminPageAttemptedLogin.jpg)
* Googling about this error leads to proposed solutions of reinstalling phpMyAdmin, probably because of misconfigurations in settings.
* Running `gobuster -e -u http://10.0.2.14/phpmyadmin -w /usr/share/wordlists/dirb/common.txt` does not yield much useful information:
![](/screenshots/lazysysadmin/gobusterPhpMyAdmin.jpg)
* `/wordpress` brings us to another site:
![](/screenshots/lazysysadmin/wordpressTR2.jpg)
* The post titled `Hello world!` is posted by user `admin`:
![](/screenshots/lazysysadmin/wordPressTR2Post.jpg)
* Since this is a WordPress site, let us try to visit `http://10.0.2.14/wordpress/wp-login.php` - which gives us our login page for this CMS! An invalid login attempt with username being `admin` confirms that the user exists.
* Running `wpscan -v --url http://10.0.2.14/wordpress` tells us that the WordPress version is `4.8.1`, and that there are 27 vulnerabilities detected:
![](/screenshots/lazysysadmin/wpscan.jpg)
* Vulnerabilities that we cannot exploit due to circumstances:
1. WordPress 3.7-5.0 (except 4.9.9) - Authenticated Code Execution: we do not have credentials to a low-level user, only `admin`.
2. WordPress 2.3-4.8.3 - Host Header Injection in Password Reset: entering `admin` when trying to `Get New Password` results in - The email could not be sent. Possible reason: your host may have disabled the mail() function.
* Note: While I got the version of `4.8.1`, the next day that I ran the same `wpscan` command resulted in the **entire list of vulnerabilities disappearing** because the detected version is now `4.8.9`. Hmm? I read from another walkthrough that the version they got is `4.8.2`.
* Running `wpscan --url http://10.0.2.14/wordpress --enumerate u` tells us that the scanner could only find one username `admin` for the CMS:
![](/screenshots/lazysysadmin/wpscanEnumUsers.jpg)
* Note: I tried `togie` as a username since there were many lines of it in the `Hello world!` post, but it does not exist.
* Detecting the plugins used reveals nothing of significance - `wpscan -v --plugins-detection aggressive --url http://10.0.2.14/wordpress`. There is only 1 plugin, `akismet` detected.
![](/screenshots/lazysysadmin/wpscanEnumPlugins.jpg)
* I ran `gobuster -e -u http://10.0.2.14/wordpress -w /usr/share/wordlists/dirb/common.txt` to see if there were unexplored files and directories but nothing interesting came up as well:
![](/screenshots/lazysysadmin/gobusterResultsWordPress.jpg)
* Note: `robots.txt` file could not be found.
* Run `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.14 http-form-post '/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Dashboard'` to attempt to get user `admin`'s password. Update: Did not manage to find any password after awhile - do not think that the password can be obtained this way.
* To-be-continued...

# SSH at Port 22
* Learning my lesson from `stapler` where I will now attempt to connect through SSH, I found that there is also a custom banner that greets us, although with not-so-useful information:
![](/screenshots/lazysysadmin/sshAttemptedLogin.jpg)
* Though, it is mentioned that this is `Web_TR1`, and we see from the WordPress site at port 80 that it is `Web_TR2`. Not sure what TR means but they are very likely to be linked.
* Attempting an SSH login later on (because I found out that `togie` could be a possible username) leads to a message that forces us to remove the offending ECDSA key. I had never seen such a message before, but I am guessing that this might be a measure to block brute-force attempts(?) Or perhaps it was simply something that I had triggered.
* `OpenSSH < 6.6 SFTP - Command Execution` does not work for us because we do not have a set of credentials (i.e. no authenticated session at all).

# NetBIOS-SSN Samba SMBD at Port 139
* Running `enum4linux 10.0.2.14` reveals that `togie` is a username. The author of this vulnerable VM shares the same name, and we had seen the same name in the `Hello world!` post on the WordPress site.
![](/screenshots/lazysysadmin/enum4linuxResults.jpg)
* At this point in time, I am not sure if there are other valuable pieces of information from the scan results (it is quite long) because this is only my second time encountering this.
* Turns out that there is - the `Share Enumeration` section. We see that there's the printer drivers, and the web server, but `Sumshare` stands out.
![](/screenshots/lazysysadmin/enum4linuxShareEnum.jpg)
* Accessing `smb://10.0.2.14/` leads us to this. We see `print$` and `share$` which were 2 sharenames in our enumeration earlier. Clicking into `share$` leads us to a login page of sorts:
![](/screenshots/lazysysadmin/smbShareLogin.jpg)
* I attempted to connect as `Anonymous` and the connection was successful! (Note: we are denied entry if we do the same for `print$`)
![](/screenshots/lazysysadmin/smbShareDirectory.jpg)
* To-be-continued...
* `exploit/linux/samba/is_known_pipename` which worked for us previously in `stapler` does not work here because there is **no writable share** here.

# MySQL at Port 3306
* This is the error encountered when we try to connect to the MySQL server, which probably explains why we see `MySQL (unauthorized)` when doing our `nmap` scan earlier.
![](/screenshots/lazysysadmin/mysqlAttemptedConnect.jpg)

# IRC at Port 6667
* Taking reference from this [article](https://www.hackingtutorials.org/metasploit-tutorials/hacking-unreal-ircd-3-2-8-1/) that I had found, I installed `hexchat` with `apt update && apt install -y hexchat`
* Next, after following the instructions to configure and connect, this is what I got:
![](/screenshots/lazysysadmin/ircConnect.jpg)
* We got the information that `InspIRCd-2.0` is running on the port.

# Concluding Remarks
1. To-be-added

# Other walkthroughs visited
1. https://grokdesigns.com/vulnhub-walkthrough-lazysysadmin-1/
2. To-be-added