---
title: Broklyn Nine Nine
date: 2023-03-19
layout: post
categories: [hack, tryhackme]
tags: [brooklyn99,ctf,tryhackme,ftp,ssh,steganography]
---

This is one of TryHackMe's easy machines, it's a lot of fun and the most special thing about this challenge is that it approaches the concept of steganography. So, let's get started!

![banner](/assets/brooklyn/banner.png)

## Enumeration

First, let's do a nmap scan on the host.

![nmap](/assets/brooklyn/nmap.png)

I ran gobuster also to check if we had other domains, but it is only the main one anyway. And this is our main page:

![website](/assets/brooklyn/page.png)

 Let's take a look at the source code of the page.

![code](/assets/brooklyn/codigo-fonte.png)

We have a clue! Do you know what is steganography? Steganography is the technique of hiding secret data within an ordinary, non-secret, file or message in order to avoid detection.

There is a project on github that does a brute force to discover the hidden data in the image. It's avaiable [here](https://github.com/Paradoxis/StegCracker).

After instalation of StegCracker, I ran this command:

`stegcracker brooklyn99.jpg`

![stegcracker](/assets/brooklyn/stegcracker.png)

Which reveals to us the word "admin". Looks like a password, but we can't connect to anything with this.

The step way is to connect to ftp with 'anonymous' login or whatever works as well.

![ftp](/assets/brooklyn/ftp.png)

We found a note to Jake! Let's take a look.

![note](/assets/brooklyn/note.png)

Interesting! I tried connecting to ssh using the credentials jake:admin, but was unsuccessful. Since the note says jake has a weak password, let's try brute force with hydra.

`hydra 10.10.250.157 -l jake -P /usr/share/wordlists/rockyou.txt ssh`

![hydra](/assets/brooklyn/hydra.png)

Perfect! We have the password!

## User Flag
Let's connect on ssh.

![user-flag](/assets/brooklyn/user-flag.png)

## Root Flag
Finally, let's stick to root flag.
I ran the command `sudo -l`

![sudo-l](/assets/brooklyn/sudo.png)

And we can see that the less command can be used as root. Let's take a look at the [GTFObins](https://gtfobins.github.io/) website to see how it works.

![less](/assets/brooklyn/sudo-less.png)

And so we got our flag.

![root-flag](/assets/brooklyn/root-flag.png)

I hope you had fun as I did too! See you in the next write-up 👋 Bye!