# DC: 5
[VulnHub link](https://www.vulnhub.com/entry/dc-5,314/)  
By DCAU

* As with the first 4 iterations of the DC-series, we are first greeted with a login page that requires users to specify both the username and the password:
![](/screenshots/dc-5/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-5/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.11`.
* The host is running 2 services, HTTP at port 80 and rpcbind at port 111. Both ports are found to be open.
* Run `nmap -p- -A 10.0.2.11`:
![](/screenshots/dc-5/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see an nginx web server running on port 80 (which is open), as with the previous iteration of the dc series.
* Opening `http://10.0.2.11` reveals a site with paragraphs of random sample texts greeting us:
![](/screenshots/dc-5/siteWebServer.jpg)
* At this point in time, I could not be more stuck. I tried using Burp Suite on the Contact form as was done in the previous walkthrough for DC-4, but to no avail.
* A hint that I got was that when I repeatedly submitted the Contact form, the copyright year at the bottom may differ:
![](/screenshots/dc-5/copyrightYears.jpg)
* It was either 2017, 2018, 2019 or 2020. Interesting. The copyright year on any of the other pages would be 2019.
* Even when I was repeatedly refreshing `http://10.0.2.11/thankyou.php` which had no parameters, the year was still changing. A contact form that is submitted would result in the URL being `http://10.0.2.11/thankyou.php?firstname=&lastname=&country=australia&subject=`.
* On hindsight, the only indication that I got from looking at the page source, is that the closing HTML tag for footer was rather out of place:
![](/screenshots/dc-5/contactFormSubmitPageSource.jpg)
* While I understood that we could modify the data (within the various options) being sent, what I did not know was that we could attempt our own options as well. Such an option would be `file`, where we would want to read the contents of specific files within the web server. It turns out that this vulnerability is called **Local File Inclusion** (LFI).
* After understanding this, we will now attempt to get the contents of `/etc/passwd`, by entering `http://10.0.2.11/thankyou.php?file=/etc/passwd`:
![](/screenshots/dc-5/etcPasswdFile.jpg)
* Interestingly, if we did not include any file to load in the parameter, the footer contents would disappear, i.e. no Copyright © 2019 would be retrieved.
* Also, `/etc/passwd` seems to be one of the common files used to test for LFI, because I guess virtually everyone would have that file on their web servers. Attempting to load a file that does not exist may lead us to be mistaken that LFI is not present.
* As much as we now know a list of users, we do not have a login page or ssh to attempt our logins.
* It turns out that to get our shell, we would have to exploit it through the *nginx access logs*.
* First, we have to find out where the access logs are located. A common location of the file is `/var/log/nginx/access.log`. Run: `http://10.0.2.11/thankyou.php?file=/var/log/nginx/access.log`:
![](/screenshots/dc-5/nginxAccessLog.jpg)
* Note: For Apache2 web servers, the location is also likely to be the same, except that it would be found under the `apache2` directory in `/var/log` instead.
* We can now confirm that we are able to retrieve the access logs, which span many, many pages.
* I accessed `http://10.0.2.11/testing-out-that-i-am-able-to-read-the-access-logs`, just to test out what we know about the access logs - that these files record all requests processed by the server.
![](/screenshots/dc-5/nginxAccessLogTest.jpg)
* Note: The latest records are found at the bottom of the log file.
* Next, run `nc 10.0.2.11 80`, then enter `GET /<?php passthru($_GET['cmd']); ?> HTTP/1.1`.
* After that, run `http://10.0.2.11/thankyou.php?file=/var/log/nginx/access.log&cmd=id` and refresh the page to confirm that we are able to run commands. We should expect to see the output of running `id` in the access logs.

# Other walkthroughs visited
1. https://bzyo.github.io/dc-5/