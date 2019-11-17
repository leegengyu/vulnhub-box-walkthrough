# Mission-Pumpkin v1.0: PumpkinFestival
[VulnHub link](https://www.vulnhub.com/entry/mission-pumpkin-v10-pumpkinfestival,329/)  
By Jayanth

## Enumeration ##
* Similar to the earlier challenges in this series, we are first greeted with a login page that requires users to specify both the username and the password. We are also given the IP address of the vulnerable machine: `10.0.2.15`.
![](/screenshots/pumpkinfestival/loginInitial.jpg)
* Because we are given the machine's IP address, we will skip the usual nmap scan of finding live hosts in our network and head straight for a more detailed scan with `nmap -p- -A 10.0.2.15`:
![](/screenshots/pumpkinfestival/hostFullScan.jpg)
* The vulnerable machine is running 3 services:
1. FTP - vsftpd 2.0.8 or later on Port 21 (the scan indicated 3.0.2 under FTP server status)
2. HTTP - Apache httpd 2.4.7 on Port 80
3. SSH - OpenSSH 6.6.1p1 Ubuntu on Port 6880
* With the exception of SSH on port 6880, the other 2 services are running on the port numbers that we expected.
* Let us first start off with exploring FTP on port 21, since we see from the nmap scan that Anonymous login is allowed.

## FTP on Port 21 ##
* First, run `ftp 10.0.2.15` and then enter `anonymous` as the username and press `Enter` when prompted for the password - and we are in! We see the directory `secret` as expected from the nmap scan. Entering the directory, we see a file `token.txt`.
![](/screenshots/pumpkinfestival/ftpLogin.jpg)
* Run `get token.txt` to download the file, and we see a string `PumpkinToken : 2d6dbbae84d724409606eddd9dd71265`. Putting it through `hash-identifier` tells us that it is either a MD5 hash or "Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))". Putting them into online hash crackers resulted in no-match.
![](/screenshots/pumpkinfestival/tokenTxt.jpg)
* There are no other files, so I guess that is all for now in this section! We will next move onto the HTTP service at port 80.

## HTTP on Port 80 ##
* Running `10.0.2.15` in our Firefox greets us with 2 potential usernames - `jack` and `harry`, and the hint that PumpkinTokens can help us to "get to our pumpkins": 
![](/screenshots/pumpkinfestival/siteHTTPPage.jpg)
* Opening up the Page Source, we see a comment that tells `harry` to "find the pumpkin", and another PumpkinToken that is hidden in the background of the page: `PumpkinToken : 45d9ee7239bc6b0bb21d3f8e1c5faa52`.
![](/screenshots/pumpkinfestival/siteHTTPPagePumpkinToken.jpg)
* [Side-note] There is a script found starting at line 25 of the Page Source that prevents right-clicking on the page.
* Opening up `robots.txt`, we see a few entries:
![](/screenshots/pumpkinfestival/robotsTxt.jpg)
* I get a `404 Not Found` on `/wordpress` and `Forbidden` on `/tokens` and `/users`. Opening up `/store/track.txt`, we get a hit which gives jack's tracking code as `2542 8231 6783 486`:
![](/screenshots/pumpkinfestival/storeTrackTxt.jpg)
* `/store` and `/img` are also `Forbidden`. Trying `/tokens/[token_string]`, `/wordpress/wp-login.php` and `/users/[user_name]` (e.g. jack / harry / admin) did not work too.
* Running a `nikto` scan did not reveal anything additional to us as well: 
![](/screenshots/pumpkinfestival/niktoScan.jpg)
* `gobuster` did not turn up anything else of note either:
![](/screenshots/pumpkinfestival/gobusterScan.jpg)
* Got a little stuck here for 4 months (this was the last machine I was attempting before school started), and I took a peek at one of the walkthroughs to move on because I did not find anything new even after revisiting this 4 months later.
* Turns out that there was indeed something I missed - a piece of information in `/store/track.txt`! I could not believe this myself - turns out that `pumpkins.local` was supposed to be inserted into `/etc/hosts`. This would then give us:
![](/screenshots/pumpkinfestival/pumpkinsLocalHomePage.jpg)
* There was really a WordPress page - I was wondering if the mention of `/wordpress/` in `robots.txt` was meant to mislead us. It turned out to be a valid finding.
* **Self-Reflection**: I think the reason why I missed this out was because on the previous occasions when I had to insert a mapping into `/etc/hosts`, there was probably a hint that I had to do so. For example, a broken page on load with the domain name being given for the broken items. This time, the page source did not give any clues for this.
* From the home page of `pumpkins.local`, we get our next token `PumpkinToken : 06c3eb12ef2389e2752335beccfb2080`.
* We are also told that pumpkins are "out of stock" and to "contact admin". Heading to `/wp-login.php`, we find that `admin` is a valid username. Both `jack` and `harry` are not valid usernames.
* Using `wpscan`, I found that I was able to access `/wp-content/uploads/` directory. There was only one file discovered from diving into it: `2019/07/1900-min.png`. We saw the photo greeting us when we loaded the home page.
* The scan also told us that we could register accounts ourselves (but I could not get anything out of it).
* There is also a `readme.html` page that returned us nothing useful: `-- This content is removed because of security purposes --`.
* Enumerating users using the scan tool showed us that there was another valid username `morse`. The third account can be ignored since I was the one who had just created it using the Register function.
![](/screenshots/pumpkinfestival/wpscanAccountEnum.jpg)
* I tried to access `/wp-config.php`, which returned me a `200 OK` blank page, the same result given when I tried to load `/wp-content` directory.
* I attempted to use `hydra` and `wpscan` to brute-force for admin and morse's credentials, but more than 30 minutes of runtime on both usernames yielded nothing so I did not pursue it.
* At this point I did not want to spend more time trying to brute-force the password, so I went back to the reference walkthrough and started from the top, seeing if I missed anything out from my earlier findings (and I did, again!).
* **Self-Reflection**: Interestingly, Pumpkin Raising had one of its password inserted into its homepage as well (`SEEDWATERSUNLIGHT`). I should have taken the cue from it.
* Turns out that `Alohomora!`, which was bolded on our very first home page, is the password to `admin`. Another slip through my earlier pair of eyes.
* After logging in, I find that I am having the `administrator` role.
* Heading to the `Posts` section, I find a draft post that gives us our next token: `PumpkinToken : f2e00edc353309b40e1aed18e18ab2c4`.
![](/screenshots/pumpkinfestival/tokenWordPressDraftPost.jpg)
* Heading to the `Pages` section, we find 6 pages, with one of the top two pages that gave us the hint of using all of our tokens to "generate our ticket to the Pumpkin Carnival".
![](/screenshots/pumpkinfestival/pageHint.jpg)
* The top 2 pages had several revisions to it - I went through all of them, thinking that I could find another token but to no avail.
* Heading to the `Users` section, I opened up the profile of `morse`, and found another token, `PumpkinToken : 7139e925fd43618653e51f820bc6201b` under the `Biological Info` section!
![](/screenshots/pumpkinfestival/morseToken.jpg)
* There was nothing under the profile of `admin`.
* Nothing else of note on the WordPress site, but we are not able to edit or add any Themes or Plugin files. Hmm.
* I remembered the hint about generating our ticket, with the phrase `Pumpkin Carnival` being bolded. I added `pumpkin.carnival 10.0.2.15` and `pumpkins.carnival 10.0.2.15` to `/etc/hosts`.
* Loading `pumpkin.carnival` on our browser led us back to the first home page that we encountered when we first entered `10.0.2.15`.
* Reaching out for another lifeline from another walkthrough, I found that there was a token that we missed under `/tokens` - `token.txt`! I tried many permutations prior to this, and this really felt like a slap for its simplicity: `PumpkinToken : 2c0e11d2200e2604587c331f02a7ebea`.
* I had also neglected a piece of information under `readme.html`, after seeing the short message regarding the removal of the content. There was a string at the next line `K82v0SuvV1En350M0uxiXVRTmBrQIJQN78s`, that did not appear to be of significance. I had thought that it was some random, useless hash code.
* Running the string on `hash-identifier` yields no results, and only after looking at the walkthrough, I discovered that it is a base-62 encoded string. We could use CyberChef (as was so in Pumpkin Raising) to decode it, or [this online decoder](https://base62.io/). The first result that turned up when I googled for "base62 decode" found the string to be an invalid base-62 one.
* Not exactly sure how one could have found it to be encoded as base62. CyberChef could not detect my string as being encoded as such, and there are 5 different baseXY encodings on CyberChef to choose from (by searching using "from base").
* After decoding it, we get `morse & jack : Ug0t!TrIpyJ`.
* There is no account `morse & jack` or `jack` in WordPress, so let us try logging in as `morse`. His password turned out to be the string indeed. This account has limited capabilities and its profile information has already been explored when we were logged in as `admin`.
* I did not know where else to use the account credentials for `jack`, because we had already covered FTP and WordPress. I had thought that the anonymous login was all to the FTP server, but trying to login with `jack:Ug0t!TrIpyJ` does not work.
* To-be-continued...

## SSH on Port 6880 ##
* Trying a login using `ssh 10.0.2.15 -p 6880`, it seems like we cannot login using a password - it must be a private key. I guess brute-forcing our way in is not an option in this case.
![](/screenshots/pumpkinfestival/sshAttemptLogin.jpg)
* Besides attempting a login as `root`, trying `jack` and `harry` also did not work.
* To-be-continued...

## Token Findings ##
* 2d6dbbae84d724409606eddd9dd71265 (FTP server)
* 45d9ee7239bc6b0bb21d3f8e1c5faa52 (first home page's page source)
* 06c3eb12ef2389e2752335beccfb2080 (second home page's page source)
* f2e00edc353309b40e1aed18e18ab2c4 (draft post)
* 7139e925fd43618653e51f820bc6201b (morse biological info)
* 2c0e11d2200e2604587c331f02a7ebea (token.txt)

## References: ##
1. https://www.hackingarticles.in/mission-pumpkin-v1-0-pumpkinfestival-vulnhub-walkthrough/