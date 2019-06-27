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
* Looking at the services which the vulnerable VM is running, there are 5 main ones - SSH at port 22, Apache HTTP server at port 80, SMB at ports 139 and 445, MySQL at port 3306 and IRC at port 6667.

# Apache Web Server at Port 80
* Visiting `http://10.0.2.14` shows us a non-standard site (powered by `Silex`), i.e. it had probably been edited from the original template - signs include "iDontCare", "AntiCare +", etc:
![](/screenshots/lazysysadmin/siteWebServer.jpg)
* Nothing quite interesting in the main page - scrolling down only tells us that the author's mind on MySQL is one of confusion, and some encouragement from the author.
* After completing the challenge: I can see why the author inserted the confusion gif for MySQL (at least from what I encountered). I mistook the error message that phpMyAdmin printed and did not know what `MySQL (unauthorized)` meant from the `nmap` scan.
* Maybe because the number of users of Silex is little, `searchsploit silex` throws up nothing.
* Opening up the `robots.txt` file shows us:
![](/screenshots/lazysysadmin/robotsTxt.jpg)
* `/old` and `/test` are empty directories, while `/TR2` is not found. The last directory, `/Backnode_files` is the only directory that gives us something - it contains the files (e.g. images) that we saw on the front page. Still, nothing quite interesting in here:
![](/screenshots/lazysysadmin/backnodeFilesDirectory.jpg)
* I ran `gobuster -e -u http://10.0.2.14 -w /usr/share/wordlists/dirb/common.txt` to find out if there were other files and directories that we could access:
![](/screenshots/lazysysadmin/gobusterResults.jpg)
* There are 3 results that stood out: `info.php`, `/phpmyadmin` and `/wordpress`.
* `info.php` shows a whole page of PHP information - not sure if there is supposed to be anything noteworthy in here:
![](/screenshots/lazysysadmin/infoPHP.jpg)
* `/phpmyadmin` page shows a login page:
![](/screenshots/lazysysadmin/phpMyAdminPage.jpg)
* Attempting a login results in an error, stating that we are unable to log in to the MySQL server because the connection failed. I guess there is no chance for us to do a wordlist attack here then.
![](/screenshots/lazysysadmin/phpMyAdminPageAttemptedLogin.jpg)
* Googling about this error leads to proposed solutions of reinstalling phpMyAdmin, probably because of misconfigurations in settings. Seems like a dead-end.
* **Continuation**: After getting a set of MySQL database credentials `Admin:TogieMYSQL12345^^` (see end of `NetBIOS-SSN Samba SMBD at Port 139` section), I realised that this error which we have been seeing does not actually mean that the database is totally down.
* Logging into phpMyAdmin as `Admin`, we see:
![](/screenshots/lazysysadmin/phpMyAdminPageSuccessfulLogin.jpg)
* However, we are not able to access any tables under `information_schema`, probably because the configurations are incomplete as seen in the warning:
![](/screenshots/lazysysadmin/phpMyAdminPageCannotViewTable.jpg)
* Running `gobuster -e -u http://10.0.2.14/phpmyadmin -w /usr/share/wordlists/dirb/common.txt` does not yield much useful information either:
![](/screenshots/lazysysadmin/gobusterPhpMyAdmin.jpg)
* Let us move onto our second result - `/wordpress`. Accessing it brings us to another site:
![](/screenshots/lazysysadmin/wordpressTR2.jpg)
* There is only one post, which is titled `Hello world!` and posted by user `admin`:
![](/screenshots/lazysysadmin/wordPressTR2Post.jpg)
* Since this is a WordPress site, let us visit `http://10.0.2.14/wordpress/wp-login.php`, which gives us our login page for this CMS. An invalid login attempt with username being `admin` confirms that the user exists.
* Running `wpscan -v --url http://10.0.2.14/wordpress` tells us that the WordPress version is `4.8.1`, and that there are 27 vulnerabilities detected:
![](/screenshots/lazysysadmin/wpscan.jpg)
* Vulnerabilities that we cannot exploit due to circumstances:
1. WordPress 3.7-5.0 (except 4.9.9) - Authenticated Code Execution: we do not have credentials to a low-level user, only `admin`.
2. WordPress 2.3-4.8.3 - Host Header Injection in Password Reset: entering `admin` when trying to `Get New Password` results in - The email could not be sent. Possible reason: your host may have disabled the mail() function.
* Note: While I got the version of `4.8.1`, the next day that I ran the same `wpscan` command resulted in the **entire list of vulnerabilities disappearing** because the detected version is now `4.8.9`. Hmm? I read from another walkthrough that the version they got is `4.8.2`.
* Note2: After getting root, I re-imported the vulnerable VM into VirtualBox, did a `wpscan`, logged in as `admin`, and both the scan and dashboard told me that the version is `4.8.1`.
* Running `wpscan --url http://10.0.2.14/wordpress --enumerate u` tells us that the scanner could only find one username `admin`:
![](/screenshots/lazysysadmin/wpscanEnumUsers.jpg)
* Note: I tried `togie` as a WordPress account username since there were many lines of it in the `Hello world!` post, but it does not exist.
* Detecting the plugins used reveals nothing of significance - `wpscan -v --plugins-detection aggressive --url http://10.0.2.14/wordpress`. There is only 1 plugin, `akismet` detected.
![](/screenshots/lazysysadmin/wpscanEnumPlugins.jpg)
* I ran `gobuster -e -u http://10.0.2.14/wordpress -w /usr/share/wordlists/dirb/common.txt` to see if there were unexplored files and directories but nothing interesting came up as well:
![](/screenshots/lazysysadmin/gobusterResultsWordPress.jpg)
* Note: `robots.txt` file could not be found.
* I ran `hydra` using the `/usr/share/wordlists/rockyou.txt` as our wordlist in an attempt to get the password of user `admin`. However, I did not manage to find any password after some time. Hence, I do not think that the password can be obtained this way.
* **Continuation**: After exploring the SMB service running on port 139, we managed to get the MySQL database credentials `Admin:TogieMYSQL12345^^`, which also allowed us to log into WordPress. Credentials are being reused here.
*: Note: It turns out that the password is pretty unique to this challenge, which explains that it was not common enough to be found on our wordlist, and thus resulting in our failed `hydra` attempt.
* From the `Dashboard` we can also confirm that the latest `wpscan` result which detected the WordPress version `4.8.9` as being correct. I am still not sure why my initial `wpscan` runs resulted in the wrong version.
* Looking at the `Users` section, we can also confirm that there is only 1 user, and that is `Admin`.
* Next, we will be uploading our reverse shell code onto one of the WordPress files. I have chosen to use the `comments.php` file, i.e. the Comments file under `Themes`. Looking at the list of files found on the right-hand side using the Themes Editor, `comments.php` is found near the top of the list.
* Copy the contents from `/usr/share/webshells/php/php-reverse-shell.php` and edit the `$ip` and `$port` to that of our Kali VM IP address and a chosen port just for the reverse shell respectively. Insert the code at the start of the WordPress file (for simplicity).
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
* To escape `rbash`, simply run the command which we have always been using to get our interactive TTY shell: `python -c 'import pty;pty.spawn("/bin/bash")'`. This was something that I had learnt from another walkthrough. However, this method would only work if `python` is installed.
* Alternatively (from what I had learnt previously), run `vi .profile` (or any other available file) to open the vi text editor first and then enter `:set shell=/bin/sh`, followed by `:shell`. Finally, run `python -c 'import pty; pty.spawn("/bin/bash")'` to get our interactive TTY shell.
![](/screenshots/lazysysadmin/escapingrBash.jpg)
* Note: In the event that we lose our current session (either by pressing CTRL + C to attempt to stop something unresponsive or otherwise), we no longer need to use `netcat` to get our shell - simply log on using SSH with `togie:12345`. Nonetheless, we would still need to break out of the `rbash` upon re-establishing the session.
* Note2: This also tells us that it was a waste of time trying to break into the admin WordPress account and establishing a shell, when we could have logged in using SSH.
* Running `cat /etc/issue; uname -a` tells us the OS version of the server, which is `Ubuntu 14.04.5 LTS`:
![](/screenshots/lazysysadmin/vulnerableOSVersion.jpg)
* At this point, while we are able to find OS exploits relating to this version of Ubuntu, they are coded in the C language and we are not able to execute the binary files that were compiled from them - the error given is `cannot execute binary file: Exec format error`. Moreover, `gcc` is not installed on the vulnerable machine.
* I was pretty stuck at this point in time, and even resorted to `sudo apt-get install gcc`.
* Little did I know that I already had the privileges to be `root` all this time!
![](/screenshots/lazysysadmin/sudoUserCommandList.jpg)
* I had run `sudo -l` earlier, but I had dismissed it because I did not know the implications of seeing `(ALL : ALL) ALL`. While I was used to seeing a single entry or two under the list of commands that our current user can run, this was my first time seeing a line full of `ALL`. I was thinking, hmm - I can run all commands as everyone - what did this mean?
* Combining the fact that `/bin/su` is among the setuid binaries that we had found earlier, and our new-found knowledge in `sudo -l`, all I had to do was run `sudo su`, and we are `root`!
![](/screenshots/lazysysadmin/sudoRootPrivEsc.jpg)
* Navigating to `/root` directory, we see the couple of usual files, but `proof.txt` is probably what we want.
![](/screenshots/lazysysadmin/rootDirectory.jpg)
* Opening up `proof.txt`:
![](/screenshots/lazysysadmin/flag.jpg)
* Not sure what the strings at the start and end of the files mean. But anyway, we are done for this challenge!

# SSH at Port 22
* Learning my lesson from `stapler`, I will now attempt to connect through SSH to see if there is any custom banner that greets us (with possibly useful information):
![](/screenshots/lazysysadmin/sshAttemptedLogin.jpg)
* Turns out there is nothing much, but it is mentioned that this is `Web_TR1`, and we see from the WordPress site at port 80 that it is `Web_TR2`. Not sure what TR means but they are very likely to be linked.
* **Continuation**: Having completed this challenge, the `Web_TR` tag that linked the two probably did not mean much - they did not have overlapping login credentials.
* Attempting an SSH login later on (because I found out that `togie` could be a possible username) leads to a message that forces us to remove the offending ECDSA key. I had never seen such a message before, but I am guessing that this might be a measure to block brute-force attempts(?) Or perhaps it was simply something that I had triggered.
* Failed attempt: `OpenSSH < 6.6 SFTP - Command Execution` does not work for us because we do not have a set of credentials (i.e. no authenticated session at all).

# NetBIOS-SSN Samba SMBD at Port 139
* Running `enum4linux 10.0.2.14` reveals that `togie` is a username.
![](/screenshots/lazysysadmin/enum4linuxResults.jpg)
* Note: Not very surprising, considering how the author of this vulnerable VM shares the same name, and we had seen the name repeatedly mentioned in the `Hello world!` post on the WordPress site.
* At this point in time, I am not sure if there are other valuable pieces of information from the scan results (it is quite long) because this is only my second time encountering this.
* **Continuation**: Turns out that there is - we should have been also looking at the `Share Enumeration` section. Under it, we see that there is the printer drivers and the web server, but `Sumshare` stands out.
![](/screenshots/lazysysadmin/enum4linuxShareEnum.jpg)
* **Alternative**: I was watching one of the walkthroughs from `ippsec` and found out that `smbmap` is his preferred tool over `enum4linux`, where the latter's output can be rather clunky (which I agree). Running `smbmap -H 10.0.2.14` gives us:
![](/screenshots/lazysysadmin/smbmap.jpg)
* `-H` stands for the IP address of the host. The output of this command alone is short and concise, and also tells us the permissions of the respective disks, which is very helpful (and would be even more so if there were a lot more disks). Moreover, I do not recall seeing these information in `enum4linux`.
* Accessing `smb://10.0.2.14/` using the Kali Linux file explorer leads us to `print$` and `share$`, which were 2 sharenames in our enumeration earlier. Clicking into `share$` results in a login page (of sorts):
![](/screenshots/lazysysadmin/smbShareLogin.jpg)
* Note: Trying to access `smb://10.0.2.14/` using a web browser such as Firefox results in a blank screen.
* I attempted to connect as `Anonymous` and the connection was successful! (Note: we are denied entry if we attempt to connect similarly without any credentials for `print$`)
![](/screenshots/lazysysadmin/smbShareDirectory.jpg)
* This is the Apache web root directory, since it contains directories such as `wordpress` and files such as `index.html`, whose contents matches what we encountered earlier when browsing through the web browser.
* The first text file that we are looking into is `deets.txt`:
![](/screenshots/lazysysadmin/deetsTextFile.jpg)
* Not sure what `CBF` means, but clearly this file was not removed as intended, and we have a potential password `12345`.
* `index.html` is the page we see when we load `10.0.2.14` on our web browser.
* `todolist.txt` reminds me of path traversal attack, since it stated **viewing** to web root using local file browser:
![](/screenshots/lazysysadmin/todolistTextFile.jpg)
* Note: This also reminds me of LFI (Local File Inclusion) where the resource is loaded **and executed** in the context of the current application. Glad I got the chance to clarify the two, because they appeared the same to me initially.
* Heading to the `wordpress` directory and opening the `wp-config.php` file, we see a set of MySQL database credentials `Admin:TogieMYSQL12345^^`:
![](/screenshots/lazysysadmin/wpConfigFile.jpg)
* But that is fine because the set of MySQL database credentials that we had just found also serves as a set of WordPress account credentials (see section on port 80)!
* Failed attempt: `exploit/linux/samba/is_known_pipename` which worked for us previously in `stapler` does not work here because this share is not writeable.

# MySQL at Port 3306
* This is the error encountered when we try to connect to the MySQL server, which probably explains why we see `MySQL (unauthorized)` when doing our `nmap` scan earlier.
![](/screenshots/lazysysadmin/mysqlAttemptedConnect.jpg)
* Even after logging in as `togie` from the same network (i.e. `10.0.2.14`), our connection is still prohibited. Hmmm.
![](/screenshots/lazysysadmin/mysqlAttemptedConnectTogie.jpg)
* Nonetheless, we managed to get access to the MySQL database through `phpMyAdmin` (see section on port 80). Hence, accessing port 3306 via an external client is probably disallowed (and it did not matter even if we were connecting from the same network).

# IRC at Port 6667
* Taking reference from this [article](https://www.hackingtutorials.org/metasploit-tutorials/hacking-unreal-ircd-3-2-8-1/) that I had found, I installed `hexchat` with `apt update && apt install -y hexchat`.
* Next, after following the instructions to configure and connect, this is what I got:
![](/screenshots/lazysysadmin/ircConnect.jpg)
* We got the information that `InspIRCd-2.0` is running on the port.
* I could not quite find an exploit for this version, even though the running version is quite old (the current released version is `3.1.0`).

# Concluding Remarks
This is the second challenge that I had attempted after `stapler`, where I felt that I was somewhat overwhelmed with the number of services running. This challenge did not present as many number of services, and is probably easier than `stapler`. Nonetheless, it gave me confidence because I am now so much better equipped, after looking back at where I had started.

Besides trying to tackle a VM with multiple services, I also wanted to try out this VM since the author's hints were something that I thought I needed more practice on. He said:
1. Enumeration is key.
2. Try harder.
3. Look in front of you.

On hindsight, his words were very true, considering how I had missed out on my final step of `sudo su`. Also, looking back at the leakage of credentials and all also made me realised that this was all in-line with the theme and name of the challenge - `LazySysAdmin`.

1. Learnt to consolidate and link information gathered from various enumeration sources.
2. Learnt how to interpret SMB enumeration results and how SMB works.
3. Learnt how MySQL works in greater detail - through interacting with it via command-line and via phpMyAdmin.
4. Learnt a new way to escape a restricted bash shell.
5. Learnt to better interpret `sudo -l` results.
6. Learnt how to connect to a IRC server using `hexchat`.

# References
1. https://grokdesigns.com/vulnhub-walkthrough-lazysysadmin-1/
2. https://windsorwebdeveloper.com/lazysysadmin1-walkthrough/