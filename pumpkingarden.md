# Mission-Pumpkin v1.0: PumpkinGarden
[VulnHub link](https://www.vulnhub.com/entry/mission-pumpkin-v10-pumpkingarden,321/)  
By Jayanth

* We are first greeted with a login page that requires users to specify both the username and the password. This is the most user-friendly GUI-based login screen that I have encountered thus far. We see the date on the top-right-hand corner and some friendly introductory text. The best part is that we are given the IP address that the machine is running on - `10.0.2.16`!
![](/screenshots/pumpkingarden/loginInitial.jpg)
* Because we are given the machine's IP address, we will skip the usual nmap scan of finding live hosts in our network and head straight for a more detailed scan with `nmap -p- -A 10.0.2.16`:
![](/screenshots/pumpkingarden/hostFullScan.jpg)
* The vulnerable machine is running 3 services:
1. FTP - vsftpd 2.0.8 or later on Port 21
2. HTTP - Apache httpd 2.4.7 on Port 1515
3. SSH - OpenSSH 6.6.1p1 on Port 3535
* Both HTTP and SSH are both running on non-conventional ports.
* Let us first start off with exploring FTP on port 21, since we already see from the nmap scan that anonymous login is allowed, and 1 file had been found.

# FTP at Port 21
* First, run `ftp 10.0.2.16` and then enter `anonymous` as the username and press `Enter` when prompted for the password - and we are in! We see `note.txt` as expected from the nmap scan.
![](/screenshots/pumpkingarden/ftpLogin.jpg)
* Run `get note.txt` to download the file, and we see a message whose only useful bit is probably the name `jack`:
![](/screenshots/pumpkingarden/noteTxt.jpg)
* Heading back into our FTP session, we find that we are already at the root directory, and that there are no other files. I guess that is all for now here! We will next move onto the HTTP service at port 1515.

# HTTP at Port 1515
* Running `10.0.2.16:1515` in our Firefox greets us with a message about a route map:
![](/screenshots/pumpkingarden/siteHTTPPage.jpg)
* Opening up the Page Source, we see a comment that says that pumpkin images would be something that can help us find a way to the route map:
![](/screenshots/pumpkingarden/siteHTTPPageSource.jpg)
* I opened the image file that we see on the loading page, and found that its URL is `http://10.0.2.16:1515/img/pumpkin.gif`.
* Going to the `/img` directory, we find a couple of images, and a `hidden_secret` directory:
![](/screenshots/pumpkingarden/imgDirectory.jpg)
* Entering the secret directory, we find `clue.txt` which contains a string `c2NhcmVjcm93IDogNVFuQCR5`. I ran `hash-identifier` against this, but not no hash type was detected.
* I tried to base-64 decode this, and found that it gave us `scarecrow : 5Qn@$y`. Seems like it is a set of login credentials. Tried to login back on FTP with them, but to no avail.
* I opened up the rest of the images and found that they were simply pumpkin images.
* There is no `robots.txt` or `readme.html` file or `/wp-login.php` page detected.
* Running a `nikto` scan did not reveal anything additional to us as well:
![](/screenshots/pumpkingarden/niktoScan.jpg)
* A `gobuster` scan was not of help either:
![](/screenshots/pumpkingarden/gobusterScan.jpg)
* Let us move onto the last service - SSH at port 3535.

# SSH at Port 3535
* Using `scarecrow:5Qn@$y`, we managed to get in. There is a `note.txt` in scarecrow's home directory - which contains 3 pieces of information: 2 usernames `LordPumpkin` (whom we understand to have root privileges) and `goblin`, as well as a secret passphrase `Y0n$M4sy3D1t`.
![](/screenshots/pumpkingarden/sshLogin.jpg)
* Trying to login as `goblin` with the passphrase (hence forming another set of credentials `goblin:Y0n$M4sy3D1t`), we find that we got in in no time! We head to goblin's home directory and find a file `note`. Opening it reveals an exploit found at `https://www.securityfocus.com/data/vulnerabilities/exploits/38362.sh` that is intended for us to use (I suppose).
![](/screenshots/pumpkingarden/goblinNote.jpg)
* Opening up exploit number `38362`'s shell script shows us that the exploit simply took advantage of `sudoedit`, which allows us to edit any file as `root`. I did not bother to run the script after initially trying to do so at `/tmp`, and instead finding the file being deleted almost instantaneously - no idea why. Moreover, we could simply make use of `sudoedit` ourselves and edit whichever files that would suit our need to be `root`.
* See **Post-Mortem** section on other methods to escalate our privileges (as should have been the intended way).
* Referring back to `dc-4`, we had 2 methods of escalating our privileges - the first of which was to add `fakeroot` to `/etc/passwd`. However, I am not sure why that did not work out for me - after adding `fakeroot::0:0:::/bin/bash` to the file, `su fakeroot` still prompted for a password.
* Hence, I used the second method where I added `* * * * * root chmod 4777 /bin/sh` to `/etc/crontab`. I checked `/bin/sh`'s permissions not too long later, and after finding that it has been given `777` permissions, I ran `/bin/sh` and volia, we are now (effectively) `root`!
![](/screenshots/pumpkingarden/privEsc.jpg)
* Heading to `/root` directory, we find `PumpkinGarden_Key`, and opening it we find a key - indicating that we have solved the challenge!
![](/screenshots/pumpkingarden/root.jpg)

# Post-Mortem - Other Methods of Privilege Escalation
* According to another walkthrough that I read, running `ps -aux` reveals to us that every 15/30 seconds, contents of the `/tmp` and goblin's `/home` directory would be removed totally, which explains why our files started disappearing really soon.
![](/screenshots/pumpkingarden/autoRemove.jpg)
* To circumvent this and use the exploit provided, we have to switch user to be `scarecrow`, then put the exploit into this user's home directory (we cannot use this user's `/tmp` directory as it would clear out too).
* Finally, create a dummy file for this exploit to edit (though I do not think this is necessary as we can simply choose any file - for example, `note.txt` in scarecrow's home directory). Give both files permissions `777`, and also the same set of permissions to the `scarecrow` directory (so goblin can access scarecrow's directory), before logging back in as `goblin`.
* Note: Trying to run it as user `scarecrow` results in this error message: scarecrow is not in the sudoers file.  This incident will be reported.
* Fun fact: While switching users, I found out that the user `jack` existed as well - we had earlier seen him being mentioned in our first finding from FTP. I suspect that this could be another 'entry point' as well.
* We head into `goblin`'s home directory, before running `./38362.sh note.txt`, allowing ourselves to now be `root`:
![](/screenshots/pumpkingarden/privEsc2.jpg)

# Concluding Remarks
This challenge was a relatively smooth-sailing one - I did not refer to any walkthroughs to get `root`, but I did get stuck for a few moments at the very last stage. By stuck, I was questioning myself why some things were not working as expected - such as the shell script being emptied virtually the very moment it was downloaded/created, and adding our own accounts to `/etc/passwd` still prompted a password.

Overall, this was a great exercise in dusting off some rust - I was away for a couple of weeks to focus on other matters and will be back for just as long as I can before school starts.

1. Learnt about `sudoedit`.
2. Re-visited `ps -aux` - I should have looked into the processes that were running to see what was causing our files in `/tmp` directory to be wiped out so quickly.

# Reference Links
1. https://medium.com/@geochris35/ethical-hacking-for-beginners-pumpkin-garden-vulnhub-walkthrough-54faebc55945