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

I log in anonymously and try to navigate directories. It looks to have only one directory, and within that directory there exists one text file, `for_Chihiro.txt`. The contents of the file reads:

```
Dear Chihiro !!

As you told me that you wanted to learn Web Development and Frontend, I can give you a little push by
showing the sources of the actual website I've created .

Normally you should know where to look but hurry up because I will delete them soon because of our
security policies !

Derry
```

We see two new users potentially: Chihiro and Derry. I looked at the sources of the pages on ports 80, 3000, and 8000 and nothing stood out as useful.

My gobusting/nikto had not finished yet, so at this point I manually looked for things like robots.txt and entering `admin:admin` as username and passwords to the Ajenti login page. Nothing useful there either.

### dirb and Nikto Results


### Gain Access
