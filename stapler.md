# Stapler: 1
[VulnHub link](https://www.vulnhub.com/entry/stapler-1,150/)  
By g0tmi1k

* We are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/stapler/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work. Also, not exactly sure why the initial login prompt and the one after an attempted login is different.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/stapler/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.18`.
* The host is running 7 services, where all of the listed ports below are open:
1. FTP at port 21
2. SSH at port 22
3. Domain at port 53
4. HTTP at port 80
5. NetBIOS-SSN at port 139
6. Doom at port 666
7. MySQL at port 3306
* There is also a closed port 20 that was running FTP-data service.
* This is our first walkthrough with so many services and ports open, probably because that there are multiple ways to get a limited shell and root access as mentioned by the author.
* Run `nmap -p- -A 10.0.2.18`:
![](/screenshots/stapler/hostFullScan.jpg)
* The scan results turned out to be really long because of the many services running.

# FTP at Port 21
* This entire section of FTP would very much likely to be brand new to me because the only time we touched on it (barely) was in `basic-pentesting-1` where we simply used a metasploit module to elevate ourselves from an unauthenticated status to a root user.
* Our `nmap` scan results showed that anonymous FTP login is allowed, and from this [article](https://shahmeeramir.com/penetration-testing-of-an-ftp-server-19afe538be4b), I found out that we can search for anonymous login permissions using this metasploit exploit:
1. `msfconsole`
2. `use auxiliary/scanner/ftp/anonymous`
3. `set rhosts 10.0.2.18`
4. `exploit`
![](/screenshots/stapler/ftpAnonymous.jpg)
* While we managed to find out that we have `READ` permissions, we did not manage to find out the vsFTPd (Very Secure File Transfer Protocol Daemon) version. The banner information was deliberately replaced with a custom message. Our `nmap` scan only told us that the version is `2.0.8` or later.
* Nonetheless, the FTP server status section under the scan did state `vsFTPd 3.0.3 - secure, fast, stable`.
* Logging into a FTP server with `ftp 10.0.2.18`, using the credentials `anonymous:anonymous`:
![](/screenshots/stapler/ftpLoginAnonymous.jpg)
* Note: Logging into a FTP server differs from that of a SSH server, i.e. we cannot run `ftp <username>@10.0.2.18`.
* `ls -al` shows us all of the files that are in the current directory, where we can find a file `note`. Turns out that `cat` does not work in FTP, and I found out that we had to use `get note` to download the file to our Kali VM:
![](/screenshots/stapler/ftpNoteFile.jpg)
* There is a message to Elly and John within `note` - not sure what it means at this point in time.
* For the next part, it appears that we have to do a dictionary attack to get a set of FTP credentials, and our dictionary wordlist would be derived by enumerating NetBIOS-SSN Samba SMBD at port 139 (see section on this below).
* We will run `hydra -L formattedwordlist.txt -P formattedwordlist.txt 10.0.2.18 ftp`:
![](/screenshots/stapler/ftpHydra.jpg)
* Note: Not sure how we would know to use the same wordlist as both username and password (because that does not seem like an intuitive thought) - I did recall seeing from the `enum4linux` results that the section started off with this line `Enumerating users using SID S-1-22-1 and logon username '', password ''`. **Did this mean that the username and password would be the same?**
* We managed to obtain 1 set of credentials: `SHayslett:SHayslett`.
* Next, let us login to the FTP server again with our set of credentials:
![](/screenshots/stapler/ftpLoginSHayslett.jpg)
* There is a long list of files in the current user's directory.
* Not sure what we should be looking out for here, so that is all here for now...

# OpenSSH at Port 22
* Interestingly, an attempted login here reveals a banner that I did not expect.
![](/screenshots/stapler/sshBanner.jpg)
* From this, we can deduce that it is very likely that there is a username `barry` in addition to `root`.
* I tried to use the `formattedwordlist.txt` from the results of `enum4linux` as the password list for a `hydra` attack, together with `barry` and `root` as the username list, but to no avail.
* Okay, so trying out `hydra -L formattedwordlist.txt -P formattedwordlist.txt 10.0.2.18 ssh` revealed the same set of credentials, `SHayslett:SHayslett`:
![](/screenshots/stapler/sshHydra.jpg)
* Again, it is interesting how the username and the password is the same.
* Logging in as `SHayslett`:
![](/screenshots/stapler/sshLoginSHayslett.jpg)
* Next, run `find / -user root -perm -4000 -print 2>/dev/null` to search for setuid binaries which we can possibly exploit:
![](/screenshots/stapler/setuidBinaries.jpg)
* Trying to list the allowed commands for `SHayslett` is forbidden:
![](/screenshots/stapler/allowedCommandsSHayslett.jpg)
* I did not know that such a restriction was possible, as we had always been successfully running this command when logged in as a user besides `www-data`.
* At this point in time, I had already attempted to access the web servers (see below), but could not manage to find any useful information. So let us go to `/var/www/https`:
![](/screenshots/stapler/webServerDirectoryContents.jpg)
* Opening up `custom_400.html`, we see that it is the page that had been greeting us (with status code 400).
* There is also an `index.html`, which states `Internal Index Page!`.
* The next file of interest is `robots.txt`, which states `Disallow: /admin112233/` and `Disallow: /blogblog/`. Let us explore those 2 respective directories.
* 

* 
![](/screenshots/stapler/htAccess.jpg)

* 

# NetBIOS-SSN Samba SMBD at Port 139
* This is even newer to me than FTP, with my only knowledge about this being that ports 137 to 139 are used for NetBIOS (Network Basic Input/Output System).
* We will use `enum4linux` here: `enum4linux 10.0.2.18`. The screenshot below only shows the information that we will be extracting from the scan - a wordlist:
![](/screenshots/stapler/enum4linuxResults.jpg)
* Note: `enum4linux` is a tool for "enumerating data from Windows and Samba hosts".
* Copy and paste the relevant result lines and paste it into a text file (e.g. `wordlist.txt`).
* Next, run `cat wordlist.txt | cut -d '\' -f2 | cut -d ' ' -f1 > formattedwordlist.txt` to just extract the dedicated string on each line, i.e. removing all redundant information.
* Interesting to learn of `cut` - my first encounter with it. The command will first remove the redundant information in the prefix, before cleaning the suffix.
* Note: `-d` is the delimiter. `-f` is the field, where the first field is `S-1-22-1-1029` as an example. Each field is separated with a whitespace (my guess after trial-and-error).
* The wordlist is very small, with only 30 lines:
![](/screenshots/stapler/wordlistFromNetBIOS.jpg)
* We will then use this wordlist to attack for a set of FTP credentials (see above section).
* `searchsploit samba` gives us a list of results:
![](/screenshots/stapler/searchsploitSamba.jpg)
* Since the Samba version that we are working with is `4.3.9`, the only one that stood out is `'is_known_pipename()' Arbitrary Module Load`.
* Heading to `msfconsole`, `search samba` reveals a shorter list, where we will `use exploit/linux/samba/is_known_pipename`.
![](/screenshots/stapler/msfconsoleSearchSamba.jpg)
* Set `RHOSTS` to the IP address of the vulnerable VM, and `RPORT` to 139:
![](/screenshots/stapler/msfconsoleOptionsSamba.jpg)
* Note: I had always thought that SMB would be run on port 445, but here it is on 139 with NetBIOS (**find out how this works**).
* And we are now `root`!
![](/screenshots/stapler/msfconsoleExploitSamba.jpg)
* Run `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn our interactive TTY shell.
* Head to `/root` directory, and we find `flag.txt`:
![](/screenshots/stapler/rootDirectoryContents.jpg)
* Here is our flag:
![](/screenshots/stapler/flag.jpg)
* The string at the end of the file `b6b545dc11b7a270f4bad23432190c75162c4a2b` is a hash, but running it on online hash crackers do not yield anything.
![](/screenshots/stapler/flagHash.jpg)

# Web Server at Port 80 and 12380
* Visiting `http://10.0.2.18` results in a Not Found error. It appears to be a PHP client server running at port 80:
![](/screenshots/stapler/siteWebServer.jpg)
* We had also found another web server running at port 12380 during our second, in-depth `nmap` scan. An Apache web server of version `2.4.18` is running at this port.
* Visiting `http://10.0.2.18:12380` results in proper site loading:
![](/screenshots/stapler/siteWebServerReal.jpg)
* It seems like there is only the default page available, because trying to load a random, non-existent section of the page resulted in the same page displayed. Same result when trying to load a file like `robots.txt`.
* The website is built by `Creative Tim`.
* I decided to try out `gobuster` instead of `dirbuster` considering its many positive reviews that I had found about it, to do a dictionary attack on the URIs (directories and files) on both web servers: `gobuster -e -u http://10.0.2.13:<PortNumber>/ -w /usr/share/wordlists/dirb/common.txt`.
![](/screenshots/stapler/gobusterResults.jpg)
* Note: `-e` lists the full path of the directory or file that is discovered.
* It turns out that we can find the `.bashrc` and `.profile` files for the web server running on port 80 - but they do not seem to reveal anything of interest.
* Running `whatweb` on port 80 did not yield any useful results, but we find the status code of `400 Bad Request` and an extra HTTP header as part of the response for the latter port:
![](/screenshots/stapler/whatweb12380.jpg)
* According to what I found from Google, a Bad Request is due to our invalid request that the server is unable to process.
* Not sure if it is due to my knowledge gap or if it is truly something that I had missed, but I do not seem to be able to find anything wrong in the request headers.
* I had also tried to send only the first 2 lines of the original request, which contains only the GET request and the host address (in case any of the other headers were invalid), but to no avail.
* `use auxiliary/scanner/http/apache_optionsbleed` on `msfconsole` (where this version of Apache HTTP server is affected by) did not work here either.

# Doom at Port 666
* Visiting `http://10.0.2.18:666/` (running service Doom, which I have no idea what it truly is) results in a page displaying unreadable information:
![](/screenshots/stapler/portDoom.jpg)

# MySQL at Port 3306
* Running on version `5.7.12-0ubuntu1`, I found an [exploit](https://www.exploit-db.com/exploits/40679) that would work on it, provided that we could get credentials to a system account.
* This [article](https://robert.penz.name/1416/how-to-brute-force-a-mysql-db/) shows us how we can do a dictionary attack using `hydra`.

# Concluding Remarks
Encountering a vulnerable machine with so many services makes things more challenging in my opinion because there appears to be so many attack vectors that we can target.

1. To-be-added

# Other walkthroughs visited
1. https://download.vulnhub.com/stapler/slides.pdf
2. https://medium.com/@Kan1shka9/stapler-1-walkthrough-e1f2a667ea4
