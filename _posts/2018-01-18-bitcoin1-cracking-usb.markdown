---
title:  "How to Lose Your Bitcoins: Part 1 (Cracking Encrypted USB Drives)"
date:   2018-01-26 12:00:00
layout: post
excerpt: "This is part one in a series of blogs on cryptocurrencies and security. The goal show how your private keys could be extracted from an encrypted USB stick, like a Tails OS persistent volume."
---

This is part one in a series of blogs on cryptocurrencies and security. The goal is to show how your private keys could be extracted from an encrypted USB stick, like a [Tails OS](https://tails.boum.org/index.en.html) persistent volume. Theses are commonly used for cold storage of Bitcoin, Ethereum, and other alt-coins.

*Please note: This is an information security blog. The intent is to help people learn about hacking, point out vulnerabilities that we encounter every day, and ultimately help people make better decisions about security. STEALING SOMEONE'S BITCOINS WOULD BE A DICKHEAD MOVE. DON'T DO THAT.*

## The Target
We're going after a [LUKS-encrypted GPT partition](https://tails.boum.org/contribute/design/persistence/), as created by the [Tails OS](https://tails.boum.org/) installer. Setting that up is outside the scope of this blog, but it's pretty easy. Full details are [available](https://tails.boum.org/install/index.en.html), but here is a quick summary:
1. Download the Tails OS iso.
2. Download the Tails OS installer.
3. Use the installer to write the ISO to a USB.
4. Boot from the USB into a live Tails environment.
5. Select "Application" > "Tails" > Configure persistent volume.
6. Set a passphrase.
7. Write some data to ~/Persistent/
8. Shut down.

## Tools
Cracking these volumes is fairly hardware-intensive, especially when using really long wordlists. For this tutorial, we are using a Linux-based computer with an Nvidia GPU.

You'll need a few things ready to go:
- [Hashcat](https://hashcat.net/hashcat/). Get the newest version from this link, some Linux package managers are woefully behind on this stuff.
- A text file that contains your encryption passphrase. While you are practicing, just make a short text file with 10 lines in it, one of them being the passphrase you set on your practice USB. Here are some good resources for cracking the real stuff:
  - My [passphrase wordlist](https://github.com/initstring/passphrase-cracker). This is a work in progress aimed at collecting common short phrases.
  - Crackstation's [wordlist](https://crackstation.net/buy-crackstation-wordlist-password-cracking-dictionary.htm). This holds more common passwords, as most people aren't really using phrases yet.
- For advanced cracking, a rule list. I like these:
  - [Hob0Rules](https://github.com/praetorian-inc/Hob0Rules).
  - [OneRule](https://github.com/NotSoSecure/password_cracking_rules).

## First Up - Find the Encrypted partition
You should be booted into your standard host OS right now - not into Tails. Insert the USB stick and wait until your computer recognizes it.

First, we need to figure out if there is an encrypted volume there at all, and if so where it is. You may get an alert right away when you plug the drive in, asking for a password. If so, just dismiss that.

Open up a terminal and use the [`lsblk`](https://www.systutorials.com/docs/linux/man/8-lsblk/) command to list all block devices. On my computer, it looks like this:

```
$ lsblk
sda                        8:0    1   3.7G  0 disk
├─sda1                     8:1    1   2.5G  0 part
└─sda2                     8:2    1   1.3G  0 part
```

Yours may not be `sda`, as that is commonly the boot volume. You should be able to easily distinguish based on the size reported.

The device `sda` has two partitions - `sda1` and `sda2`. This was created by the Tails Installer, which we happen to know uses the first partition for the boot volume and the second partition for encrypted persistent storage. To verify, we can run the following command and you should see output similar to mine:

```
$ sudo cryptsetup luksDump /dev/sda2
LUKS header information for /dev/sda2

Version:       	1
Cipher name:   	aes
Cipher mode:   	xts-plain64
Hash spec:     	sha256
Payload offset:	4096
MK bits:       	256
MK digest:     	b7 e5 d9 40 35 52 03 1a 42 3d a1 49 8f a3 0f 59 af cd 68 04
MK salt:       	3b 5a 3b 42 4b 88 7d 63 b7 dd 16 d7 7e 51 47 04
               	de 47 92 32 f2 f6 53 a7 fb 4e dd 07 6a c2 56 34
MK iterations: 	351500
UUID:          	2f2fde5c-0a41-4f1d-9e08-b013a94c1edf

Key Slot 0: ENABLED
	Iterations:         	2763832
	Salt:               	63 99 a0 26 9c 26 58 0c f2 4e 5b a6 04 70 67 03
	                      	76 3f 25 01 4b 40 2b 09 d8 b8 d3 33 77 54 8a ad
	Key material offset:	8
	AF stripes:            	4000
Key Slot 1: DISABLED
Key Slot 2: DISABLED
Key Slot 3: DISABLED
Key Slot 4: DISABLED
Key Slot 5: DISABLED
Key Slot 6: DISABLED
Key Slot 7: DISABLED
```

Awesome, that's what we're looking for. You can see specifics on the encryption methods used in "Cipher name", "Cipher mode", "Hash spec", etc.

*Don't get too excited - I created this drive especially for this tutorial, so don't hope to snatch any coins by anything you see here. :)*

If you run this same command against a non-encrypted volume, you'll see something like this:

```
$ sudo cryptsetup luksDump /dev/sda1
Device /dev/sda1 is not a valid LUKS device.
```

## Next - Clone the Encrypted Volume
USB drives can be volatile, with many of them being cheap pieces of crap that will lose your data at the worst time possible. Do **NOT** try to recover encrypted data directly from a USB drive.

We want to make a local copy of our target, `/dev/sda2` on our cracking machine. This is easy accomplished using [`dd`](https://www.systutorials.com/docs/linux/man/1-dd/). The following command will create a file called "crypt.img" in the local directory.

```
$ sudo dd if=/dev/sda2 of=./crypt.img status=progress
```

Once that completes, you can use the same method of checking for an encrypted volume we used above. The new command should like this:

```
$ sudo cryptsetup luksDump ./crypt.img
```

The output should look the same as when you ran it directly against the USB stick. Remove that stick now, to make sure you don't accidentally delete those sweet sweet coins.

## Get Cracking - Standard Dictionary Attack
Now comes the fun. Cracking passwords is an art, and consistent success requires really fine tuning your approach. If you want to learn some advanced methods, Google around a bit or check out [this great book](https://www.amazon.com/gp/product/B075QWTYPM).

Assumptions:
- You've downloaded hashcat and placed the files into `/opt/hashcat`.
- You've configured your drivers as recommended [here](https://hashcat.net/wiki/doku.php?id=linux_server_howto). This is important.
- You've created a short text file, with one potential password per line, your password being one of them. That text file is at `~/wordlist.txt`
- The `crypt.img` file is also in your homefolder, at `~/crypt.img`.

First, we are going to run a straight-up dictionary attack. This means that password has to be found in your wordlist exactly - with correct case, special characters, etc.

Here we go:

```
# Try it this way first, with some hardware optimization parameters:
$ /opt/hashcat/hashcat64.bin -a 0 -m 14600 ~/crypt.img ~/wordlist.txt -O -w 3

# If that doesn't work, try this:
$ /opt/hashcat/hashcat64.bin -a 0 -m 14600 ~/crypt.img ~/wordlist.txt
```

Press the `S` key at any time to see that status of your cracking session.

If your session completes successfully, you should see an output with your password. If the session completed and you aren't sure it was successful, running the command as follows will show you all successfully cracked passwords for a given target:

```
$ /opt/hashcat/hashcat64.bin -a 0 -m 14600 ~/crypt.img --show
```

If the output of the above command is blank, the password has not yet been cracked.

## Getting Tricky - Rule-Based Attacks
As humans, we are pretty dumb when it comes to making passwords. We think doing things like adding an `!` or replacing an `S` with a `$` makes them more secure.

When it comes to password cracking, this may slow us down a bit but certainly doesn't stop us. We can use a pre-defined set of rules for transforming files in a wordlist to many possible permutations.

If you'd like to test this out, use a password like `ThisPasswordSucks!` for your USB stick, but in the wordlist you create enter only `ThisPasswordSucks`. This isn't really stressing your rule list, but you get the idea.

Assumptions:
- You've downloaded [Hob0Rules](https://github.com/praetorian-inc/Hob0Rules) and placed it in `/opt/rules/`.

Follow the same process as before, but alter the command to use one of the rule sets as follows:

```
$ /opt/hashcat/hashcat64.bin -a 0 -m 14600 ~/crypt.img ~/wordlist.txt -r /opt/rules/Hob0Rules/hob064.rule -O -w 3
```

## Happy Hacking!
Have fun with this! If you have any questions or need any help, feel free to reach out.
