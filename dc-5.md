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
![](/screenshots/dc-4/siteWebServer.jpg)
* Clicking on the various sections of the page on the bar near the top of the page brings us to more random sample texts, with the exception of `Contact` where there is a form (i.e. user inputs) and a submit button.
* The `robots.txt` is not found (404) and the page source gives us nothing of interest, neither does running `uniscan`.
* However, we do know that the nginx version which is running here is `1.6.2`, which is pretty much older than the `1.15.10` version in DC-4. There might be exploits relating to this version or older which we can use.
* To-be-continued...