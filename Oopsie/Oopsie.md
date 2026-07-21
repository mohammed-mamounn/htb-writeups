# Oopsie | Machine | Very Easy | Linux

## Introduction
Let's start with reconnaissance.

```
nmap <IP> -sC -sV
```

Ports 80 (http) & 22 (ssh) are open. Starting with SSH is out of the question because I don't have any lead as to what the password could be.

So I tried visiting the website, & honestly there was nothing special.

## Web Proxying with Burp Suite
A web proxy is an intermediary server that sits between you and the internet. Your request goes to the proxy first, which fetches the page from the destination server and returns it to you, effectively hiding your identity (IP address for example) from that server

Burp Suite is the industry-standard platform used for web app security testing. It is essentially an interception proxy sitting between our web browser & the web server, allowing us to view, analyze & modify HTTP/S traffic in real time.

- Burp Suite uses port 8080 by default for its proxy listener.

So I tried disabling the interception of Burp & configuring the proxy of our browser to 127.0.0.1 on port 8080. I refreshed the page which mapped multiple directories in Burp. The most interesting one was `/cdn-cgi/login` which took me to a login page. This seemed like the most viable option for now.

After navigating through the dashboard, I came across the directory holding uploads.

A Web Crawler is a program or automated script that browses the web methodically.

## Cookie Manipulation
A cookie is a text file with data created by the web server & stored by the browser on your computer to identify a session.

I noticed on top, the search bar had "user_id". I found the cookies used a "role" & "user_id" value which are manipulatable, and it turns out they are! After editing both, I decided to change the user_id to look for other users (`user=2` & role `admin`).

Now for the foothold, I tried uploading a PHP reverse shell.

I started my listener `nc -lvnp 1234` & immediately requested `http://<ip>/uploads/php-reverse-shell.php` & it worked.

I upgraded the shell via `python3 -c "import pty; pty.spawn('/bin/bash')"`.

Now it was time for lateral movement since I can't do anything as `www-data`.

I started enumerating the directories & found `/var/www/html/cdn-cgi/login`, & decided to look for passwords used.

`cat * | grep -i passw` reads all files & pipes/feeds the output to `grep` searching for "passw" (`-i` means case-insensitive).

## Privilege Escalation
I got a password, `MEGACORP-4dm!n[all]`.

I checked available users with `cat /etc/passwd` & "robert" seemed like a good candidate.

I tried `su robert`, but unfortunately the password didn't work.

So I kept reading around & found the user "will" — the password `gs4CO.rPUs3r` worked, and I got the user flag.

I next used `ls -la /usr/bin/bugtracker` to check privileges & type of file.

The result was a file called `/usr/bin/bugtracker` with the SUID bit set.

SUID (Set owner User ID) is a special permission on a file — a file with SUID always executes as the OWNER of the file, regardless of who executes the command.

`s` (lowercase) = file owner has execute permission (SUID set & executable)
`S` (uppercase) = SUID set but NOT executable

To escalate our privileges we first need to know what we're allowed to do:

`sudo -l` → not allowed to run sudo.

`id` → shows which groups we belong to, including the `bugtracker` group.

`find / -group bugtracker 2>/dev/null` → helps locate files owned by that group.

I ran the app `/usr/bin/bugtracker`. It calls `cat` on a file.

The owner of the binary is root.

## PATH Hijacking
Since we're unprivileged, `./bugtracker` relies on the `$PATH` variable to locate the `cat` binary — it doesn't use a hardcoded path like `/bin/cat`, it just searches `$PATH`.

Since `bugtracker` runs as root (SUID) and searches `$PATH` to find `cat`, if we prepend `/tmp` to `$PATH`, it will find our malicious `cat` in `/tmp` first & execute it as root — giving us a root shell.

```
cd /tmp
echo "/bin/bash" > cat
chmod +x cat
export PATH=/tmp:$PATH
```

You might ask why we didn't create the malicious `cat` in `/bin` instead — that's because we don't have write privileges there; we can only read in `/bin`.

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
