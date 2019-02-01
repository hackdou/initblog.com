---
title:  "Cracking Passwords Based on Song Lyrics"
date:   2017-04-26
---

*Update (2019): I've since released a much more powerful tool for cracking passphrases. Check it out [on my Gitlab page here](https://gitlab.com/initstring/passphrase-wordlist). I'll keep this old blog up as the output from lyricpass can be fed through the cleanup.py script and then used with the hashcat rules in the new project. Enjoy!*


There’s been a lot of news in the media lately about using tools like encryption and password managers. Both of these can leverage a single password to unlock a ton of vital information. Because of this, people are looking to create longer, more complex “master keys”. This blog demonstrates a method of guessing some of those keys.

These “master keys” are for things like:

- Unlocking their password manager
- Decrypting their OS when booting
- Decrypting volumes with sensitive files
- WiFi encryption using WPA
- etc.

I’ve noticed a trend when people are discussing new strategies here – using a string of words together, including spaces, that is easy to remember. Song lyrics come readily to mind, and it seems that a good deal of people assume these will be difficult to crack.

## So What?

I wanted to create a short program to show that this type of password is also insecure. Using Python with a few simple libraries, I created [this script](http://github.com/initstring/lyricpass) that generates a password list based on a given artist. Discovering someone’s favorite band is pretty easy... that sort of thing is plastered all over social media, and it’s usually something people will provide when asked by anyone.

```
$ python ./lyricpass.py -h
usage: lyricpass.py [-h] [--lower] [--punctuation] artist output

positional arguments:
  artist         Define a specific artist for song lyric inclusion. Please
                 place the artist name in quotes.
  output         Output to file name in current directory.

optional arguments:
  -h, --help     show this help message and exit
  --lower        Switches all letters to lower case.
  --punctuation  Preserves punctuation, which is removed by default.
  ```

The script currently allows a few inputs to tailor the password file, which is pure text and can be used for brute force password attacks. It will chug away and output a nice deduplicted file when it is done.

![screenshot](/images/post-lyricpass/1.png)

Are you using any lyric-based passwords today? Try out [this script](http://github.com/initstring/lyricpass) and see if it can crack yours. 

## Update
This script is now linked to on [WeakPass](https://weakpass.com/links)! :)
