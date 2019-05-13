# HackInOS: 1
[VulnHub link](https://www.vulnhub.com/entry/hackinos-1,295/)  
By Fatih Çelik

* As with [Basic Pentesting: 1](https://github.com/leegengyu/CTF-Walkthrough/blob/master/basic-pentesting-1.md), after the victim VM has been booted up, we are greeted with a login page that requests for the password of user `hummingbirdscyber`. We attempt a few common passwords such as `password` and `admin`, but we do not manage to login as expected.
* The IP address of the victim VM can be found by clicking the icon with 2 arrows pointing up and down respectively on the top right-hand corner of the login page, which is `10.0.2.15`.
![](/screenshots/hackinos-1/vulnerableVMIPAddress.jpg)
* From our Kali VM, run `nmap -p- -A 10.0.2.15`, scanning all ports, and enabling OS detection, version detection, script scanning, and traceroute:
![](/screenshots/hackinos-1/scanAllPortsandServiceVersions.jpg)
* There are 2 ports that are open: 22 (ssh) and 8000 (http). Typically, the HTTP service is run on port 80, but I guess the point of having it on a different port on this vulnerable VM is to test us.
* At this point, I am not sure if there is a vulnerability on the SSH service that we can exploit. Hence, we will go ahead to explore the web service first.

# HTTP service #
* We visit `10.0.2.15` and find a web server running, with the rendered page broken:
![](/screenshots/hackinos-1/httpServicePage.jpg)
* Hovering our mouse over the `Log in` hyperlink under the `Meta` section shows us that the domain name is expected to be localhost running on port 8000. instead of 10.0.2.15. `localhost` does not resolve to 10.0.2.15, but to 127.0.1.1 instead (see below) hence the broken page.
![](/screenshots/hackinos-1/httpServicePageHover.jpg)
* To resolve this issue, edit our `/etc/hosts` file accordingly:
![](/screenshots/hackinos-1/hostsFileEntries.jpg)
* Note: The commented-out line for the localhost entry that maps to 127.0.1.1 is deliberate so that we can add back this entry after we are done with this challenge, since that was the original entry in the file.
* Reload `10.0.2.15` to see the intended layout of the page. There is only 1 post in the entire WordPress site by user `Handsome_Container`:
![](/screenshots/hackinos-1/wordPressProperLoad.jpg)
* We see that there is a comment on the only post, but the comment reveals nothing of concern:
![](/screenshots/hackinos-1/wordPressPost.jpg)
* Head to the `/wp-login.php` page to confirm that there is indeed a WordPress account with the username `Handsome_Container`. At the same time, we find out that there is no account with the username `admin`.
