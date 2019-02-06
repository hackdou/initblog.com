---
title:  "How to Steal Bitcoins: Part 2 (Cracking Bitcoin Core wallet.dat Files)"
date:   2018-02-05 12:00:00
published: true
layout: post
excerpt: "This is part two in a series of blogs on cryptocurrencies and security. The goal is to recover passwords from encrypted Bitcoin Core (or Satoshi Client) wallets."
---

This is part two in a series of blogs on cryptocurrencies and security. The goal is to recover passwords from encrypted Bitcoin Core (or Satoshi Client) wallets.

*Please note: This is an information security blog. The intent is to help people learn about hacking, point out vulnerabilities that we encounter every day, and ultimately help people make better decisions about security. STEALING SOMEONE'S BITCOINS WOULD BE A DICKHEAD MOVE. DON'T DO THAT.*

## The Target
We're making progress! Following the instructions in [part one](/2018/bitcoin1-cracking-usb/), you've gained access to an encrypted USB drive. Looking around, you notice a some interesting files - perhaps in a hidden folder called `.bitcoin`. Inside that folder, look for a file called `wallet.dat` - that contains all of the information needed to access a bitcoin wallet and its associated funds.

You can also search for other `.dat` files, as these can be manual backups created by the user.

Let's assume you've already tried to import the file into a wallet application and are prompted to enter a password. If you haven't yet tried this - do it now. You might get lucky.

![Locked!](/images/post-btc2/locked.png)
*Nah, that would have been too easy!*



## Tools
Cracking these wallets can be fairly hardware-intensive, especially when using really long wordlists. For this tutorial, we are using a Linux-based computer with an Nvidia GPU.

You'll need a few things ready to go:
- [Bitcoin2john](https://raw.githubusercontent.com/magnumripper/JohnTheRipper/bleeding-jumbo/run/bitcoin2john.py). This is a fork of pywallet modified to extract the password hash in a format that hashcat can understand.
- [Hashcat](https://hashcat.net/hashcat/). Get the newest version from this link, some Linux package managers are woefully behind on this stuff.
- A text file that contains your encryption passphrase. While you are practicing, just make a short text file with 10 lines in it, one of them being the passphrase you set on your practice wallet. Here are some good resources for cracking the real stuff:
  - My [passphrase wordlist](https://github.com/initstring/passphrase-cracker). This is a work in progress aimed at collecting common short phrases.
  - Crackstation's [wordlist](https://crackstation.net/buy-crackstation-wordlist-password-cracking-dictionary.htm). This holds more common passwords, as most people aren't really using phrases yet.
- For advanced cracking, a rule list. I like these:
  - [Hob0Rules](https://github.com/praetorian-inc/Hob0Rules).
  - [OneRule](https://github.com/NotSoSecure/password_cracking_rules).

## First Up - Extract the Password Hash
In this step, we'll extract a hash that can be used to crack the password. Place both `bitcoin2john.py` and `wallet.dat` in the same folder. Inside the terminal, run the python script as follows:

```bash
python ./bitcoin2john.py ./wallet.dat
```

Take the line that starts with `$bitcoin` and place it in a file called `hash.txt` in the working directory.

![Hash](/images/post-btc2/hash.png)

**Note**: If you're reading this because you've forgotten your own password and are unable to crack it yourself, you can share this hash with a reuptable wallet recovery service. Cracking this hash will not allow them to access your Bitcoins unless they also have access to your `wallet.dat` file. However, it you've re-used this password elsewhere, it may allow them to log in to those services and otherwise make your life hell.


## Get Cracking - Standard Dictionary Attack
Now comes the fun. Cracking passwords is an art, and consistent success requires really fine tuning your approach. If you want to learn some advanced methods, Google around a bit or check out [this great book](https://www.amazon.com/gp/product/B075QWTYPM).

Assumptions:
- You've downloaded hashcat and placed the files into `/opt/hashcat`.
- You've configured your drivers as recommended [here](https://hashcat.net/wiki/doku.php?id=linux_server_howto). This is important.
- You've created a short text file, with one potential password per line, your password being one of them. That text file is at called `wordlist.txt` and is in the working folder with your `hash.txt` file.

First, we are going to run a straight-up dictionary attack. This means that password has to be found in your wordlist exactly - with correct case, special characters, etc.

Here we go:

```bash
# Try it this way first, with some hardware optimization parameters:
/opt/hashcat/hashcat64.bin -a 0 -m 11300 ./hash.txt ./wordlist.txt -O -w 3

# If that doesn't work, try this:
/opt/hashcat/hashcat64.bin -a 0 -m 11300 ./hash.txt ./wordlist.txt
```

Press the `S` key at any time to see that status of your cracking session.

If your session completes successfully, you should see an output with your password. If the session completed and you aren't sure it was successful, running the command as follows will show you all successfully cracked passwords for a given target:

```bash
/opt/hashcat/hashcat64.bin -a 0 -m 11300 ~/hash.txt --show
```

If the output of the above command is blank, the password has not yet been cracked.

## Getting Tricky - Rule-Based Attacks
As humans, we are pretty dumb when it comes to making passwords. We think doing things like adding an `!` or replacing an `S` with a `$` makes them more secure.

When it comes to password cracking, this may slow us down a bit but certainly doesn't stop us. We can use a pre-defined set of rules for transforming files in a wordlist to many possible permutations.

If you'd like to test this out, use a password like `ThisPasswordSucks!` for your `wallet.dat`, but in the wordlist you create enter only `ThisPasswordSucks`. This isn't really stressing your rule list, but you get the idea.

Assumptions:
- You've downloaded [Hob0Rules](https://github.com/praetorian-inc/Hob0Rules) and placed it in `/opt/rules/`.

Follow the same process as before, but alter the command to use one of the rule sets as follows:

```
/opt/hashcat/hashcat64.bin -a 0 -m 11300 ./hash.txt ./wordlist.txt  -r /opt/rules/Hob0Rules/hob064.rule -O -w 3
```

![Cracked!](/images/post-btc2/btc-cracked.png)
*As you can see, this password sucks. I hope the hash above didn't get you too excited.*

## Happy Hacking!
Have fun with this! If you have any questions or need any help, feel free to reach out.

And seriously - don't steal any bitcoins that don't belong to you. We all want this stuff to be mainstream one day, and that isn't going to happen if people don't feel safe using them.
