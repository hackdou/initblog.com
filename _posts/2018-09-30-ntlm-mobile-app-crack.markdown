---
title:  "Hijacking NTLM-powered Mobile Apps (Part 1 - Cracking with Responder)"
date:   2018-09-30 12:00:00
published: true
layout: post
excerpt: "Doing a black-box test of a mobile app that uses NTLM authentication to speak to the web service? You may find your typical tools won't work. Read on for information on intercepting, inspecting, and modifying the API calls. You might even get lucky and crack a clear-test master password."
---

Doing a black-box test of a mobile app that uses NTLM authentication to speak to the web service? You may find your typical tools won't work. Read on for information on intercepting, inspecting, and modifying the API calls. You might even get lucky and crack a clear-test master password.

*Please note: This is an information security blog. Please don't do anything stupid without permission.*

## The Problem
When I look at mobile applications, the juiciest data tends to be not on the physical device but instead hosted online. The mobile device will speak with that target via an API, at some point negotiating the level of access that the device should be provided. Before that negotiation occurs, the device needs to prove itself worthy of having that conversation. This is often done with some sort of API key or key/secret combo.

Once we plug the mobile device into our intercepting proxy and bypass and TLS restrictions, we can usually see that conversation happening. But what happens when the app stops functioning completely once you enable the proxy?

If this happens to you, look for the following:

- In the "Dashboard" tab of Burp Suite, under "Event log": Do you see "Authentication failure from xxx"?
- If you browse to that domain name in your web browser, does it go right to a 401 - Unauthorized? Or maybe even pop a logon box? If so, enter some random credentials and then view the response in Burp Suite.

Take a close look at these headers:

```
HTTP/1.1 401 Unauthorized
Content-Type: text/html
Server: Microsoft-IIS/7.5
WWW-Authenticate: NTLM
X-Powered-By: ASP.NET
Date: Mon, 01 Oct 2018 00:26:54 GMT
Content-Length: 850
```

The interesting item is the `WWW-Authenticate` line. This is the server asking us to come back to it with a [Challenge/Response](https://docs.microsoft.com/en-us/windows/desktop/SecAuthN/microsoft-ntlm) conversation to authenticate using a valid Windows account.

This is a problem. The challenge/response conversation is [connection-oriented](https://msdn.microsoft.com/en-us/library/cc236629.aspx), meaning that we can't pick it apart as individual HTTP requests inside Burp. It's sort of like the SSL handshake that occurs before we see any requests intercepted - it either works or it doesn't. If you happen to have the username and password that the mobile account is using to speak to the web server, [Burp can handle that](https://support.portswigger.net/customer/portal/articles/2927576-configuring-ntlm-with-burp-suite). But, in a black-box test, we don't and we are stuck with nothing but a 401.

There's two things we can do, which I'll split into separate blog posts:
1. Intercept the NTLM Challenge/Response, crack it to clear text, and use the credentials in Burp
2. Password won't crack? Relay the NTLM Challenge/Response to the web service, crafting your own API calls

Intercepting NTLM challenge/response is a widely known attack method, but you usually hear about it used on local networks with SMB connections. Using [Responder](https://github.com/lgandx/Responder), we can intercept one of these requests and obtain a hash for cracking. Here is the basic flow for getting this done in a mobile app.

## Set up a trusted certificate

Your app probably uses TLS, so we need a trusted certificate to use with Responder. You can use my [certificate cloning tool here](https://github.com/initstring/pentest/blob/master/web-tools/clone-ssl.py) to make a certificate that is sort-of like your target domain. This probably won't bypass certificate pinning (unless it is done really badly), so you may have some more steps on your own to get past that. Assuming your 401 request is going to `https://google.com/mobileapp/veryvuln.svc` you will run the tool like this:

```
$ ./clone-ssl.py google.com
[+] Got the certificate...
[+] Parsed the certificate...
[+] Made the fake CA cert...
[+] Made the fake client cert...
[+] Wrote the following files to clonessl-output:
    * ca.crt
    * ca.key
    * ca.pem
    * client.crt
    * client.key
    * client.pem
```

Take the `ca.crt` file and get it onto the iOS device. I like to run `python -m 'SimpleHTTPServer'` from the folder where I just created the certificates, and then browse to http://mycomputerip:8000 from the iOS device. When you click on the `ca.crt` file you will get a message saying "This website is trying to open Settings to show you a configuration profile. Do you want to allow this?". Click "Allow".

You should see some shady certificate details that look sort-of like the CA of your actual target. Click the "Install" button in the top right. You should get a couple more warnings, allow them all and click "Done" once you see a green check mark.

The profile is now installed, but that CA certificate is not trusted. There is one more task on iOS:
- Go to the "Settings" app
- Choose "General"
- Choose "About"
- Choose "Certificate Trust Settings" (at the bottom)
- Toggle the switch to the right for the CA we just installed.

## Take control of DNS
We want to send the request to our own machine - spoofing DNS is a great way to do this. I like to use this [hotspot script](https://github.com/oblique/create_ap) to run an access point from a USB wifi device on my laptop. Then, connect your mobile device to that hotspot. If you run the script with the `-d` parameter, it will look in your local /etc/hosts file for records before querying an external DNS. So, add an entry for the IP address of your computer and the DNS domain name of your target BEFORE running the hotspot script. The commands might look something like this:

```
# Add a host file entry (192.168.12.1 is the IP that create_ap assigns my access point)
echo "192.168.12.1      google.com" >> /etc/hosts

# Create an AP using wlan01 and bridging to eth0 for Internet access
sudo ./create_ap wlan01 eth0 myssid SuperComplexPassword -d
```

Just as a sanity check, make sure things are working. Run the following command on your computer:

```
sudo nc -nlvp 443
```

And open up the mobile app, trying to do something that would query the API. You should see see a connection received, like this:

```
Listening on [0.0.0.0] (family 0, port 443)
Connection from <some IP> 54618 received!
```

If you get this message, let's keep going. If not, go back and troubleshoot.

## Run a modified version of Responder

Now, we want to run Responder on the same interface. I went through Hell getting it to work with a particular mobile app. I had to use a [very specific version](https://github.com/SpiderLabs/Responder/releases/tag/v2.3.0) of Responder - basically the last one from Spiderlabs before Laurent forked it off to his own repo. I also had to modify the initial 401 reply that the tool sends back to the client. It needed something more than just the headers in the reply, otherwise there was no challenge/response.

You can can make the change yourself like this:
- Edit servers/HTTP.py
- Look at line 219: `Response = IIS_Auth_401_Ans()`
- Change it to something like this:

```
                        Response = ("HTTP/1.1 401 Unauthorized\r\n"
                                    "Content-Type: text/html\r\n"
                                    "Server: Microsoft-IIS/7.5\r\n"
                                    "WWW-Authenticate: NTLM\r\n"
                                    "X-Powered-By: ASP.NET\r\n"
                                    "\r\n"
                                    "Unauthorized\r\n\r\n")
```


OK, assuming you have Responder in /opt/Responder and have made the change above:

```
# Replace the generic Responder certs with those we created from my tool above
cp clonessl-output/client.crt /opt/Responder/certs/responder.crt
cp clonessl-output/client.key /opt/Responder/certs/responder.key

# Run Responder on the same interface you are using to host the wireless hotspot
# Don't forget the -v flag, it helps troubleshoot NTLM stuff
sudo /opt/Responder/Responder.py -I wlan0 -v

```

If you want to do another sanity check here, you can open up Safari on the mobile device and browse to https://targetdomain. The DNS should resolve to the machine running Responder, and you should get a logon box popping up. Just as important, you should NOT be getting any certificate errors if you set up that stuff properly. If this looks good, time to move to the app.

Now, just open the app on your mobile device and do something that should trigger communication with the API. If everything is working, you will be rewarded with the lovely sight of the NTLM hash in the console.

## Crack the hash

Pop that bad boy in a text file called `responder-hash` and take a run at it with hashcat as follows:

```
hashcat -a 0 -m 5600 ./responder-hash dictionary-file -r rule-file
```

Make sure to check out [my passphrase cracking project](https://github.com/initstring/passphrase-wordlist) as well.

## Configure Burp with your new creds

If you were lucky enough to crack the password to clear text, you can now configure Burp with the username and password as follows:

- Go to "Project Options"
- Under "Platform Authentication" check "Overrise user options" and "Do platform authentication"
- Click "Add"
- Fill in the details for your target host
- Set "Authentication Type" to (probably) NTLMv2
- Fill in username/password/domain

You can now intercept and modify commands as you would any other mobile app. The app developer probably never intended this, so you may find some nice juicy vulnerabilities in the API.

Happy Hacking!

## Can't crack the password?

If you find your GPU heating the house to no avail, fret not! We can still take that Challenge/Response and relay it on a per-request basis, allowing us to manually craft API calls outside of the app developers good intentions.

Read [part 2](/2018/ntlm-mobile-app-relay/) to learn more...
