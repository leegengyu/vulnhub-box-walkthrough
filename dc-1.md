# DC: 1
[VulnHub link](https://www.vulnhub.com/entry/dc-1,292/)  
By DCAU

* As with [Mr-Robot: 1](https://github.com/leegengyu/CTF-Walkthrough/blob/master/mr-robot-1.md), we are first greeted with a login page that requires users to specify both the username and the password: 
![](/screenshots/dc-1/LoginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-1/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.7`.
* The host is running SSH (port 22), HTTP (port 80) and rpcbind (port 111) services, where the ports for these 3 servers are open.
* Run `nmap -p- -A 10.0.2.7`:
![](/screenshots/dc-1/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, there is an Apache httpd web server running on port 80, with the website being run by a `Drupal CMS`.
* Moreover, there appears to be a long list of results from the `robots.txt` file as well.
* Open `10.0.2.7` on our browser and we have the welcome page greeting us with a login panel on the left-hand side:
![](/screenshots/dc-1/SiteWebServer.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* I tried a couple of passwords on the account with the username `admin`, with my attempts being spaced out over 3 hours, but I was greeted with this message on the most recent attempt:
![](/screenshots/dc-1/AccountTemporarilyBlocked.jpg)
* Note: I found this [documentation page](https://www.drupal.org/node/1023440) by Drupal that states its policy on blocking logins after 5 failed login attempts, and how it is being recorded.
* I guess that means that we can rule out brute-forcing attempts to get an account access.
* At the same time, I found that the login page did not reveal any hints about whether the account (username) existed, with a failed attempt resulting in this error message:
![](/screenshots/dc-1/invalidLogin.jpg)
* However, if you get the account-temporarily-blocked message as shown above, or if you go to `Request new password` and attempt to recover your password, you would be able to deduce if the particular user existed.
* For the latter, these are the 2 different messages that you encounter when you submit the username or email address of a valid user vs. invalid user:
![](/screenshots/dc-1/AdminUsername.jpg)
* The page source is somewhat long (compared to the other ones that I have seen thus far from the 3 other vulnerable VMs) at around 100-150 lines. However, I could not quite seem to find any noteworthy information from it.
* Visiting `robots.txt`, we see a long list of disallow directories, files and paths:
![](/screenshots/dc-1/robotsTxt.jpg)
* I did not attempt to visit all of them, but from my quick sampling, the directories forbidded access, while the pages gave an access-denied message too.
* I went on to find out more about the CMS that ran the site, which is Drupal who is running on version 7. This was interesting for me because the previous vulnerable VMs were all running on WordPress.
* I ran a quick Google search and found that Drupal 7 had various vulnerabilities that we could exploit.
* To further identify what the version that the Drupal CMS was running on, we will be using `droopescan` ([GitHub link](https://github.com/droope/droopescan)).
* Note: It appears that `droopescan` comes packaged with the Kali VM, as seen [here](https://en.kali.tools/all/?tool=371). I could not find it on mine, hence I will be installing it below:
* After cloning the Git repository, enter the local folder and run `pip install -r requirements.txt`, where `-r` allows to refer to other requirement files as well.
* Run `droopescan scan drupal -u http://10.0.2.7`:
![](/screenshots/dc-1/droopescanResult.jpg)
* We can see that the possible versions of the Drupal CMS is narrowed down to 7.22 - 7.26, though we do not have an exact hit.
* Running `msfconsole`, I searched for modules pertaining to Drupal: `search drupal`:
![](/screenshots/dc-1/msfconsoleSearchResults.jpg)
* We can either `use exploit/multi/http/drupal_drupageddon` or `use exploit/unix/webapp/drupal_drupalgeddon2`.
* According to Rapid7, the [former module](https://www.rapid7.com/db/modules/exploit/multi/http/drupal_drupageddon) exploits the Drupal HTTP Parameter Key/Value SQL Injection (aka Drupageddon) in order to achieve a remote shell on the vulnerable instance. It was tested against Drupal 7.0 and 7.31 (fixed in 7.32).
* The [latter module](https://www.rapid7.com/db/modules/exploit/unix/webapp/drupal_drupalgeddon2) exploits a Drupal property injection in the Forms API. Drupal 6.x, < 7.58, 8.2.x, < 8.3.9, < 8.4.6, and < 8.5.1 are vulnerable.
* Whichever module that we use, we would first `show options` to see what module option settings is required of us to set.
* There are many parameters in both of the modules, but we would only have to edit RHOST (which is left empty initially): `set RHOST 10.0.2.7`.
* Run show options again to confirm that we have the correct parameters set:
![](/screenshots/dc-1/drupalDrupageddon.jpg)
![](/screenshots/dc-1/drupalDrupalgeddon2.jpg)
* Run `exploit` (not showing 2 screenshots here because running the exploit command for both results in the same output):
![](/screenshots/dc-1/exploit.jpg)
* Enter `sysinfo` to confirm that we have a reverse shell on the right machine and then enter `shell`: we find that we are now user `www-data`.
* Next, we need a privilege escalation in order to get `root` and before continuing further, we will spawn an interactive TTY shell first: `python -c 'import pty; pty.spawn("/bin/bash")'`.
* We will next be finding a list of binaries that have the setuid bit enabled on the vulnerable VM: run `find / -user root -perm -4000 -print 2>/dev/null`.
![](/screenshots/dc-1/suidExecutables.jpg)
* Out of all the binaries returned as part of our `find` command, we will be using `/usr/bin/find`.
* Normally, we will use the `find` command to search for files, but we will not be specifying any file names in our command (and it will not throw out the entire list of available files as well). Instead, we will use the `-exec` parameter, which will execute the command to spawn a shell, which turns out to give us `root`.
* Run `find -exec "/bin/sh" \;`:
![](/screenshots/dc-1/findCommandPrivilegeEscalation.jpg)
* How the command is constructed is that all following arguments to `find` are taken to be arguments to the command until an argument consisting of `;` is encountered. `\` is the escape character here to protect the semi-colon from expansion by the shell. This explanation is taken from the [Linux manual page](http://man7.org/linux/man-pages/man1/find.1.html)
* Note: Using `/bin/bash` instead will result in a non-root shell. This has nothing to do with the fact that we spawned an interactive TTY `/bin/bash` shell earlier on. I tried to set the TTY shell to `/bin/sh`, but having `/bin/bash` at this point still results in a non-root shell. I suspect that this has to do with existing .bash setting files.
* Note: You can also run the same `find` command with an existing file name (typed out in file including the file extension). The directory that I was in had a bunch of files, the first of which is `COPYRIGHT.txt`. Just add this file name after `find` and before `-exec`. However, this command will not work with a non-existent file name, or if the file extension is missing.
* We are now `root`. To get our final flag, we head to `/root`, and open `thefinalflag.txt`:
![](/screenshots/dc-1/thefinalflag.jpg)
* Note: Running `exit` to terminate our root shell seems to have no effect at all. I could only send a terminate signal (CTRL + C) to the channel, which sends us straight back to our meterpreter terminal.
* Hurray, that is the end of this challenge!