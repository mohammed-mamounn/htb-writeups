# Fawn | Machine | Very Easy | Linux

## Introduction

File Transfer Protocol is a service that runs on port 21, think of it as a digital shipping service between 2 computers.

Plain FTP does not encrypt data, FTPS adds encryption & SFTP routes all commands through a secure shell tunnel. FTP has this anonymous log-in mechanism that allows users to enter without knowing the password (helloooo), however they can only read & download, not modify.

## Methodology

Started by running the good old nmap, discovered port 21 is open and the anonymous function of FTP is enabled here, so I wrote the command ```ftp```, then ```open 10.129.89.229```,  I was then asked for credentials (username & password).
After typing anonymous and leaving the password field empty I got the code `230` login successful!

Afterwards, I tried listing the content of the server using `ls` and sure senough, I found the ```flag.txt```!


## What I Learned
- Anonymous FTP is a surprisingly common misconfiguration on easy machine, it's always worth trying `anonymous`/blank password before assuming a service needs credentials.
- Plain FTP sends everything, including any login credentials, in cleartext. FTPS and SFTP exist specifically to fix this by adding encryption or tunneling through SSH.
- Even a "read-only" anonymous share can leak sensitive files if no one audits what's actually placed inside the accessible directory, that's exactly how the flag was exposed here.
- Basic FTP commands (`ls`, `get`) are enough to fully enumerate and pull files off a server once you're in.
- Small, low-effort misconfigurations (leaving anonymous login on, forgetting a sensitive file in a public directory) can be just as exploitable as a "real" vulnerability, this machine didn't need any exploit code at all.

## Remediation
- Disable anonymous FTP access entirely unless there's a clear need for it.
- If anonymous access must stay enabled, restrict it to an isolated directory with nothing sensitive in it.
- Make the anonymous share strictly read-only.
- Replace plain FTP with FTPS or SFTP so traffic isn't sent in cleartext.
- Regularly audit what files are reachable from the FTP root, it's easy for a stray backup or sensitive file to end up somewhere exposed, which is exactly what happened here.
- Log and monitor FTP sessions to catch unusual access patterns.
