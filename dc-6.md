# DC: 6
[VulnHub link](https://www.vulnhub.com/entry/dc-6,315/)  
By DCAU

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
* Run `hydra -l admin -P passwords.txt wordy http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Dashboard'.
* It turns out that admin's password is not within the list. Some other user's password is likely to be within it though. Run `wpscan --url http://wordy --enumerate u`:
![](/screenshots/dc-6/wordPressUserList.jpg)
* There are 4 users whom we were unaware of previously: `sarah`, `graham`, `mark` and `jens`. Let us add them into `usernames.txt`.
* Now, we will run `hydra` again, but without attempting to get admin's password: hydra -L usernames.txt -P passwords.txt wordy http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Dashboard'.
![](/screenshots/dc-6/hydraPasswordResults.jpg)
* We managed to get a set of credentials: `mark:helpdesk01`.
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
* To-be-continued...

# Concluding Remarks
To-be-added

# Other walkthroughs visited
To-be-added