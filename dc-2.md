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
* Looking at the services which the vulnerable VM is running, we can see an Apache httpd web server running on port 80 (which is open), with the `http-title` stating that there is a redirect to `http://dc-2` which it did not follow.
* There is also an open OpenSSH service running on port 7744 (which is open) which was not detected earlier in the `nmap` scan.
* Open `http://10.0.2.7` on our browser and the address is indeed redirected to `http://dc-2`, but the page still fails to load.
* Note: The `http://` prefix cannot be left out in this case, as compared to our previous attempted vulnerable VMs.
* To solve this issue, add the entry `10.0.2.7 dc-2` to `/etc/hosts`. The key idea here is that for the page to properly load, the requests must reach the web server at the IP address. Since the IP address redirects us to `http://dc-2` initially, what we have done here is to ensure that the latter resolves to the former during the DNS resolution process.
* Opening `http://10.0.2.7` or `http://dc-2` now reveals a WordPress site, with 5 sections for us to explore:
![](/screenshots/dc-2/siteWebServer.jpg)
* All of the sections contain sample texts with nothing noteworthy except the `Flag` section, which gives us `Flag 1`:
![](/screenshots/dc-2/flag1.jpg)
* The site which holds `Flag 1` is `http://dc-2/index.php/flag/`.
* Clicking on any of the 5 sections or the main page does not give us the links to any of the other locations on the site, hence I tried something like `http://dc-2/index.php/admin`, which gave us an error message stating that the page cannot be found and at the same time allowing us to use the search function:
![](/screenshots/dc-2/wordPressMissingPage.jpg)
* It does not matter what you type into the search field, but hitting the search button allows us to get to this page which shows us what posts there are on the WordPress page, as well as the link to the login site:
![](/screenshots/dc-2/wordPressSearch.jpg)
* We see from the only post on the site, titled `Hello world!` that the post was published by the user `admin`. Besides that, there appears to be nothing noteworthy from the post and the comment to the post.
![](/screenshots/dc-2/wordPressPost.jpg)
* Heading to the login page at `http://dc-2/wp-login.php`, we confirm that the username `admin` exists, but common login passwords such as `admin` and `password` do not work. `Flag 1` mentioned that our "usual wordlists probably won't work", and was probably referring to the use of wordlists on this login page.
* I decided to test out what Flag 1 said about our usual wordlist not working, by using the wordlist `/usr/share/wordlists/rockyou`. After about 7,500 attempts into the wordlist, I stopped it since I was not seeing any results.
* Flag 1 said that we "just need to be `cewl`", and initially I thought that it actually meant that we needed to be `cool`. I searched up `cewl` and found it to be a "ruby app which spiders a given url to a specified depth, optionally following external links, and returns a list of words which can then be used for password crackers", according to the [Kali tools](https://tools.kali.org/password-attacks/cewl) page.
* There is an example of `cewl` usage on the Kali tools page: `cewl -d 2 -m 5 -w docswords.txt https://example.com`.
* Using the example as a basis, I ran `cewl -w passwords.txt http://dc-2`, ommiting the `-d` and `-m` options because their default values of `-d 2` meant that the tool would spider to a depth of 2 and `-m 3` meant that the minimum word length is 3. These default values appear to suffice for a start, and we will adjust them accordingly later on if required.
* I opened `passwords.txt` and as expected, the words within the word list were compiled from different parts (also depending on parameter set) of the site, which explains why it was mentioned that `cewl` "spiders a given url to a specified depth".
* Running `wc -l passwords.txt` tells us that there were 263 different passwords in the list.
* Since we have a list of possible passwords now, we will search if there are any other users besides `admin`. Moreover, I re-read flag 1 and the last sentence said that if we cannot find it, log in as another. I assumed that this meant that there were other accounts on the WordPress site besides the `admin` one (and that getting the password for `admin` is likely to be difficult), but we have to find these other accounts through other means because the site only revealed the admin account earlier on.
* We will be using `wpscan --url http://dc-2 --enumerate u`:
![](/screenshots/dc-2/wpscanUsersResults.jpg)
* It turns out that there are a total of 3 accounts: `admin`, `jerry` and `tom`, where we were not aware of the latter 2.
* Next, I ran `hydra -L users.txt -P passwords.txt dc-2 http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Location'` to brute-force the respective passwords for the 3 users.
![](/screenshots/dc-2/hydraCrackPasswords.jpg)
* It seems that we were not able to find the password for user `admin`, but the password for user `jerry` is `adipiscing`, and the password for user `tom` is `parturient`.
* Note: We can also use `wpscan` to brute-force the respective passwords, though the trade-off is that the latter is only able to brute-force one user's password at a time, as compared to `hydra` which can brute-force multiple users' passwords using the same wordlist.
* I was curious where jerry and tom's password was located on the WordPress site, and found the former immediately on the `Welcome` page - turns out that it was on the very first line of what greeted us. tom's password is found on the `Our Products` page, in paragraph 4.
![](/screenshots/dc-2/wordPressUserjerryPasswordLocation.jpg)
* After logging in separately as user `jerry` and `tom`, we find that their accounts are not of the Administrator type, given the limited options we see on the left-hand side toolbar, and also from the `Profile` page:
![](/screenshots/dc-2/wordPressUserjerryAndtomProfile.jpg)
* Navigating the various sections on the left-hand side toolbar reveals no significant information except the `Pages` section, where we see `Flag 2` is located.
![](/screenshots/dc-2/flag2Location.jpg)
* Opening up the Page in edit mode gives us `Flag 2`:
![](/screenshots/dc-2/flag2.jpg)
* From the details of the post, the Page was published, but it probably did not appear on the site (where `Flag 1` did) because it was not inserted on the front page. The URL of the published Page is: `http://dc-2/index.php/flag-2/`.
* Turns out that we could have just obtained `Flag 2` by modifying the url of `Flag 1`. I immediately tried `/flag-3/`, `/flag-4/`, `/flag-5/` and `/flag-final/` but they were all not found (or at least not viewable as the user `jerry` or `tom`).

* We will now explore the other service, i.e. SSH that is also running on the vulnerable VM. Since we have the WordPress login credentials to `jerry` and `tom`, we will likewise attempt a SSH login using those same credentials: `ssh [username]@dc-2 -p 7744`.
* We are able to successfully log in with `tom:parturient`:
![](/screenshots/dc-2/sshtomLogin.jpg)
* However, we can also see that we are stuck with a `rbash`, i.e. a restricted shell that restricts some features of bash shell.
* We are still able to run commands such as `ls`, which shows us that `Flag 3` is in the current working directory:
![](/screenshots/dc-2/flag3Location.jpg)
* However, we are not able to view the flag via commands such as `cat` and `more`.
* Fortunately, the `vi` text editor is available to us, and opening up `flag3.txt` gives us the following:
![](/screenshots/dc-2/flag3.jpg)
* Note: I tried other text editors such as `vim` and `nano` but they were not available.
* To get out of `rbash`, I executed `:set shell=/bin/sh` and `:shell` within `vi`, as learnt earlier from an OverTheWire Bandit [exercise](https://github.com/leegengyu/OverTheWire-Bandit) at level 25.
* Note: Setting the shell to `/bin/bash` does not work. Also, `!sh` for the latter command does not work.
* After getting out of our `rbash`, we find that some commands still do not work, because they are "not found":
![](/screenshots/dc-2/binshCommandNotFound.jpg)
* To resolve this issue, we have to execute `export PATH=/usr/sbin:/usr/bin:/sbin:/bin`, which restores the $PATH variable.
* Going back to interpreting `Flag 3`, we find that the hint given is to run `su jerry`, since we are currently `tom`.
* After running the command, we enter jerry's password `adipiscing`, and find that we have successfully logged in as `jerry`:
![](/screenshots/dc-2/sshjerryLogin.jpg)
* After logging in, we head to tom's home directory and find `Flag 4`:
![](/screenshots/dc-2/flag4Location.jpg)
* Opening up `flag4.txt`, we find that the hint given is on the very last line - that we have to use `git`.
* Running `sudo -l`, we try to get a list of commands that we are able to execute (and forbidden from executing) as `root`:
![](/screenshots/dc-2/gitCommand.jpg)
* Run `sudo git help add` to enter the `Git Manual` page of the `git-add` command, and then enter `!/bin/sh`:
![](/screenshots/dc-2/binshCommandGit.jpg)
* Note: Instead of the parameter `add`, you can also use any of the `git-` commands, so long as we manage to enter a git manual page.
* Note: Instead of running `/bin/sh`, you can also run `/bin/bash`.
* Hurray, we have now gotten `root` access!
* We head to the `root` directory, and find `final-flag.txt`:
![](/screenshots/dc-2/flagLast.jpg)