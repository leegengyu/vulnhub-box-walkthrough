# DC: 2
[VulnHub link](https://www.vulnhub.com/entry/dc-2,311/)  
By DCAU

* As with [DC: 1](https://github.com/leegengyu/CTF-Walkthrough/blob/master/dc-1.md), we are first greeted with a login page that requires users to specify both the username and the password: 
![](/screenshots/dc-2/loginInitial.jpg)
* Common login credentials such as `admin:admin` and `admin:password` do not work.
* Run `nmap 10.0.2.*`, where we find 5 live hosts:
![](/screenshots/dc-2/nmapScan.jpg)
* After eliminating our own IP address and our router gateway IP address, it is very likely that the vulnerable VM's IP address is `10.0.2.7`.
* The host is running only the HTTP (port 80) service, where the port for this server is open.
* Run `nmap -p- -A 10.0.2.7`:
![](/screenshots/dc-2/hostFullScan.jpg)
* Looking at the services which the vulnerable VM is running, we can see a Apache httpd web server running on port 80, with the `http-title` stating that there is a redirect to `http://dc-2` which it did not follow.
* Open `http://10.0.2.7` on our browser and the address is indeed redirected to `http://dc-2`, but the page still fails to load.
* Note: The `http://` prefix cannot be left out in this case, as compared to our previous attempted vulnerable VMs.
* To solve this issue, add the entry `10.0.2.7 dc-2` to `/etc/hosts`. The key idea here is that for the page to properly load, the requests must reach the web server at the IP address. Since the IP address redirects us to `http://dc-2` initially, what we have done here is to ensure that the latter resolves to the former during the DNS resolution process.
* Opening `http://10.0.2.7` or `http://dc-2` now reveals a WordPress site, with 5 sections for us to explore:
![](/screenshots/dc-2/siteWebServer.jpg)
* All of the sections contain sample texts with nothing noteworthy except the `Flag` section, which gives us `Flag 1`:
![](/screenshots/dc-2/flag1.jpg)
* The site which holds `Flag 1` is `http://dc-2/index.php/flag/`.
* Clicking on any of the 5 sections or the main page does not give us the links to any of the other locations on the site, hence I tried something like `http://dc-2/index.php/admin`, which gave us an error message stating that the page cannot be found and at the same time allowing us to use the search function.
![](/screenshots/dc-2/wordPressMissingPage.jpg)
* It does not matter what you type into the search field, but hitting the search button allows us to get to this page which shows us what posts there are on the WordPress page, as well as the links to the login site.
![](/screenshots/dc-2/wordPressSearch.jpg)
* We see from the only post on the site, titled `Hello world!` that the post was published by the user `admin`. Besides that, there appears to be nothing noteworthy from the post and the comment to the post.
![](/screenshots/dc-2/wordPressPost.jpg)
* Heading to the login page at `http://dc-2/wp-login.php`, we confirm that the username `admin` exists, but common login passwords such as `admin` and `password` do not work. `Flag 1` mentioned that our "usual wordlists probably won't work", and was probably referring to the use of wordlists on this login page.
* I decided to test out what Flag 1 said about our usual wordlist not working, by using the wordlist `/usr/share/wordlists/rockyou`. About 7,500 attempts into the wordlist I stopped it since I was not seeing any results.
* Flag 1 said that we had to be `cewl`, and initially I thought that it actually meant that we needed to be `cool`. I searched up `cewl` and found it to be a "ruby app which spiders a given url to a specified depth, optionally following external links, and returns a list of words which can then be used for password crackers", according to the [Kali tools](https://tools.kali.org/password-attacks/cewl) page.
* I ran `cewl -d 2 -m 5 -w docswords.txt http://dc-2` initially, following the exact same set of parameters given in the example on the Kali tools page.
* Next, I ran `wpscan --url http://dc-2 --usernames admin --passwords docswords.txt` to brute-force the password for user `admin`. However, that did not work and I changed the parameters to `-d 5 -m 10`. Still, there were no valid passwords found.
* I re-read flag 1 and the last sentence said that if we cannot find it, log in as another. I assumed that this meant that there were other accounts on the WordPress site besides the `admin` one, but we have to find it through other means because the site only revealed the admin account.
* To do so, we run: `wpscan --url http://dc-2 --enumerate u`:
![](/screenshots/dc-2/wpscanUsersResults.jpg)
* It turns out that there are a total of 3 accounts: `admin`, `jerry` and `tom`, where we were not aware of the latter 2.
* I re-ran `wpscan --url http://dc-2 --usernames jerry --passwords docswords.txt` to brute-force the password for user `jerry`, using the wordlist `docswords.txt` with a minimum word length of 10:
![](/screenshots/dc-2/wordPressUserjerryPassword.jpg)
* And we have the password `adipiscing` for user `jerry`! I guess we were pretty lucky with this set of credentials because I was just trying random numbers for the minimum password length parameter.
* After logging in as user `jerry`, we find that the account is not of the Administrator type, given the limited options we see on the left-hand side toolbar, and also from our `Profile` page:
![](/screenshots/dc-2/wordPressUserjerryProfile.jpg)
* Navigating the various sections on the left-hand side toolbar reveals no significant information except the `Pages` section, where we see `Flag 2` is located.
![](/screenshots/dc-2/flag2Location.jpg)
* Opening up the Page in edit mode gives us `Flag 2`:
![](/screenshots/dc-2/flag2.jpg)
* From the details of the post, the Page was published, but it probably did not appear on the site (where `Flag 1` did) because it was not inserted on the front page. The url of the published Page is: `http://dc-2/index.php/flag-2/`.
* Turns out that we could have just obtained `Flag 2` by modifying the url of `Flag 1`. I immediately tried `/flag-3/`, `/flag-4/`, `/flag-5/` and `/flag-final/` but they were all not found (or at least not viewable as the user `jerry`).
