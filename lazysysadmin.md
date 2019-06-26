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
* **Continuation**: After getting the MySQL database credentials `Admin:TogieMYSQL12345^^` below, I realised that this error which I have been seeing does not actually mean that the database is down.
* Logging into phpMyAdmin as `Admin`, this is what we see:
![](/screenshots/lazysysadmin/phpMyAdminPageSuccessfulLogin.jpg)
* However, we are not able to access any tables under `information_schema`, probably because the configurations are incomplete as seen in the warning:
![](/screenshots/lazysysadmin/phpMyAdminPageCannotViewTable.jpg)
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
* Continuation: After exploring the SMB service running on port 139, we managed to get the MySQL database credentials `Admin:TogieMYSQL12345^^`, which also allowed us to log into WordPress. It turns out that the password is pretty unique to this challenge, which explains that it was not common enough to be found on a wordlist and thus resulting in our failed `hydra` attempt.
* From the `Dashboard` we can also confirm that the latest `wpscan` result which detected the WordPress version `4.8.9` as being correct. I am still not sure why my initial `wpscan`s resulted in the wrong version.
* Looking at the `Users` section, we can confirm that there is only 1 user, and that is `Admin` ourselves.
* Next, we will be uploading our reverse shell code onto one of the WordPress files. I have chosen to use the `comments.php` file, i.e. Comments file under `Themes`. Looking at the list of files using the Themes Editor, `comments.php` is found near the top of the list.
* Copy the code from `/usr/share/webshells/php/php-reverse-shell.php`, edit the `$ip` and `$port` to that of our Kali VM IP address and a chosen port just for the reverse shell respectively. Insert the code at the start of the WordPress file (for simplicity).
![](/screenshots/lazysysadmin/themesReverseShellCode.jpg)
* After successfully editing the file, make sure that we have our `netcat` listener running on our Kali VM: `nc -v -l -p 1234`.
* Next, load the web page that would trigger our reverse shell code to execute. In my case, I will simply load the comments page of the `Hello world!` post, i.e. `http://10.0.2.14/wordpress/?p=1#comments`.
* And we have our shell! Run `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn our interactive TTY shell, and `export TERM=xterm` to enable commands such as `clear` for our convenience. As expected, we are user `www-data`:
![](/screenshots/lazysysadmin/netcatShell.jpg)
* Heading to `/home`, we find that there is a user `togie`, but the password `TogieMYSQL12345^^` is incorrect.
* Next, run `find / -user root -perm -4000 -print 2>/dev/null` to search for setuid binaries which we can possibly exploit. Turns out that there appears to be nothing of interest though:
![](/screenshots/lazysysadmin/setuidBinaries.jpg)
* Failed attempt: The `unix-privesc-check` script is not able to run because `strings` is not installed.
* I remembered next that we had a password `12345` from `deets.txt` in `share$` sharename. I tried `su togie`, with `12345` being the password. We are now user `togie`:
![](/screenshots/lazysysadmin/suTogie.jpg)
* Trying to navigate out of our current directory led us to realise that we are now stuck within a restricted bash shell:
![](/screenshots/lazysysadmin/rbashRestricted.jpg)
* To escape `rbash`, simply run the command which we have always been using to get our interactive TTY shell: `python -c 'import pty;pty.spawn("/bin/bash")'`. This was something that I had learnt from another walkthrough. However, this would only work if `python` is installed.
* Alternatively (what I had learnt previously), run `vi .profile` (or any other file) to open the vi text editor first and then enter `:set shell=/bin/sh`, followed by `:shell`. Finally, run `python -c 'import pty; pty.spawn("/bin/bash")'` to get our interactive TTY shell.
![](/screenshots/lazysysadmin/escapingrBash.jpg)
* Note: In the event that we lose our current session (either by pressing CTRL + C to attempt to stop something unresponsive or otherwise), we no longer need to use `netcat` to get our shell - simply log on using SSH with `togie:12345`. Nonetheless, we would still need to break out of the `rbash` upon re-establishing the session.
* Running `cat /etc/issue; uname -a` tells us the OS version of the server, which is `Ubuntu 14.04.5 LTS`:
![](/screenshots/lazysysadmin/vulnerableOSVersion.jpg)
* At this point, while we are able to find OS exploits relating to this version of Ubuntu, they are coded in the C language and we are not able to execute the binary files that were compiled from them - the error given is `cannot execute binary file: Exec format error`. Moreover, `gcc` is not installed on the vulnerable machine.
* I was pretty stuck at this point in time, and even resorted to trying to attempt `gcc` with `sudo apt-get install gcc`.
* I simply did not understand that I had the privileges to go to root at this point in time!
![](/screenshots/lazysysadmin/sudoUserCommandList.jpg)
* While I was used to seeing a single entry or two under the section of the commands that our current user can run, this was my first time seeing `(ALL : ALL) ALL`. I was thinking, hmm, I can run all commands as everyone - what did this mean?
* All I had to do was run `sudo su`, and we are `root`!
![](/screenshots/lazysysadmin/sudoRootPrivEsc.jpg)
* There are the couple of usual files in the `/root` directory, but `proof.txt` is probably what we want.
![](/screenshots/lazysysadmin/rootDirectory.jpg)
* Opening up `proof.txt`:
![](/screenshots/lazysysadmin/flag.jpg)
* And we are done for this challenge!

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
* I attempted to connect as `Anonymous` and the connection was successful! (Note: we are denied entry if we attempt to connect similarly for `print$`)
![](/screenshots/lazysysadmin/smbShareDirectory.jpg)
* The first text file that we are looking into is `deets.txt`. Not sure what `CBF` means, but clearly this file was not removed as intended, and the password is not likely to have been updated as well (I am guessing). Anyhow, we will take note of the password `12345`.
![](/screenshots/lazysysadmin/deetsTextFile.jpg)
* `index.html` is the page we see when we load `10.0.2.14` on our web browser.
* `todolist.txt` reminds me of path traversal attack, since it is stating **viewing** to web root using local file browser.
* Note: This also reminds me of LFI (Local File Inclusion) where the resource is loaded **and executed** in the context of the current application. Glad I got the chance to clarify the two, which appeared the same to me initially.
![](/screenshots/lazysysadmin/todolistTextFile.jpg)
* Heading to the `wordpress` directory and opening the `wp-config.php` file, we see the set of MySQL database credentials `Admin:TogieMYSQL12345^^`, as well as the authentication unique keys and salts:
![](/screenshots/lazysysadmin/wpConfigFile.jpg)
* Having explored the `share$` sharename, we are reminded that we do not have write permissions (only read) to any of the content within.
* But that is fine because the set of MySQL database credentials that we just found, also works for the WordPress login (see section on port 80)!
* Failed attempt: `exploit/linux/samba/is_known_pipename` which worked for us previously in `stapler` does not work here because there is **no writable share** here.

# MySQL at Port 3306
* This is the error encountered when we try to connect to the MySQL server, which probably explains why we see `MySQL (unauthorized)` when doing our `nmap` scan earlier.
![](/screenshots/lazysysadmin/mysqlAttemptedConnect.jpg)
* Even after being logged in as `togie`, our connection from `10.0.2.14` is prohibited as well. Hmmm.
![](/screenshots/lazysysadmin/mysqlAttemptedConnectTogie.jpg)
* We managed to get access to the MySQL database through `phpMyAdmin` (see section on port 80). Hence, accessing port 3306 via an external client is probably disallowed, and it did not matter even if we were connecting from the same network.

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