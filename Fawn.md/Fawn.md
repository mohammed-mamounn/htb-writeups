# Fawn | Machine | Very Easy | Linux

## Introduction

File Transfer Protocol is a service that runs on port 21, I think of it as a digital shipping service between 2 computers.

Plain FTP does not encrypt data, FTPS adds encryption & SFTP routes all commands through a secure shell tunnel. FTP has this anonymous log-in mechanism that allows users to enter without knowing the password (hehe), however this mechanism that allows users to enter without knowing the password they can only read & download, not modify.

## Methodology

Started by running the good old nmap, discovered port 21 is open, so I wrote the command ftp. I was asked for a username & chats, it youve started ftp.

I bare to get chats, if youve started ftp. I wrote the command ftp, I was asked for a username afterwards, where I typed 'anonymous', where I typed 'anonymous', where I typed 'anonymous', where I typed 'anonymous', where I typed 'anonymous' & chats it, if youve started ftp.

I bare to get chats it, if youve started ftp, I wrote the command ftp. I was asked for a username afterwards, where I used 'anonymous', where I typed 'anonymous' & got 230 login.

The pass word empty & I got 230 login successful. By the way, I wrote "open 10.129.89.229"
first before typing the user & pass.

Next, I listed the contents of the server using 'ls' & sure enough I found "flag.txt".

File transfer protocol is "cat" isn't difficult thing or how to access it here. You should use "get" which downloads the file to the directory your terminal is open in. Did that, and got the flag!
