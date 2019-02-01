---
title:  "Hacking a Pizza Order with Burp Suite"
date:   2017-09-03
---

Web hacking skills can be used to solve critical challenges in business and life – like customizing a pizza order. Read on to see how I overcame a restricted UI to triumphantly top my pizza just the way I wanted it.

The issue was this – I’d gone online to order a pizza on a Saturday evening. When I went to customize the order, I noticed that the special cheese I wanted was not listed as an option. This was a problem. Once the mental switch has been flipped on pizza, there is simply no going back. It’s pizza or die.

I’d moved houses recently, and apparently the location near my new address didn’t offer the same toppings as their other store. I called their number to ask what the deal was, and the person on the phone insisted I should be able to order it online.

![1](/images/post-pizza/1.jpg)

First, I opened a new private browsing tab in Firefox. I configured the same pizza from the store near my old address. As you can see, the list of toppings at each location varies slightly:

![2](/images/post-pizza/2.jpg)

OK, so I’m not crazy. Well, you could probably make a case that ordering that particular ingredient and putting it on a pizza is far from sane, and I wouldn’t argue that. But at least my memory was correct – the other location did indeed offer this ungodly option.

At this point, I fired up my trusty Burp Suite and configured Firefox to proxy the requests through it. Burp Suite is an application that allows us to inspect and modify HTTP requests – you can read more about it in my last post.

Using the browser tab for the old pizza location, I selected my desired ingredients and clicked on the "Update" button. This triggered the following HTTP POST request, as showing in Burp:

![3](/images/post-pizza/3.png)

Interesting. It seems that numbers are being sent for two relevant parameters – *"currIngreds"* and *"extIngreds"*. The former changes when removing default toppings and the latter changes when adding extra toppings.

I imagine that each topping is given a unique numerical value that allows the sum to be translated into the individual parts, sort of like file permissions in Linux.

Well, like any HTTP POST request, we are the masters of our own destiny here. While the UI for the new location does not allow me to create this combination, I can most certainly modify the input I send them to try to replicate the desired outcome.

From here, I turn the “intercept” function on – this allows me to modify the POST request before it is submitted. I edit the pizza toppings at the new location, simply ticking random boxes. When the request comes to Burp Suite, I modify *"currIngreds"* and *"extIngreds"* to exactly match the screenshot above.

![4](/images/post-pizza/4.png)

After selecting "forward" in Burp, my browser presents me with the listing shown above – the impossible order has been created successfully!

![5](/images/post-pizza/5.jpg)

This is an interesting real-world example of how a UI restricting user input doesn’t necessarily keep savvy users from doing what they want. While my particular modification was benign, similar strategies can be used to modify more critical elements in some web apps – like price.

During a penetration test, this is a key strategy to keep in mind. Enumerate how the application behaves, and you may be able to apply a successful outcome from one area to another. Web developers should also pay attention to and prepare for this type of behavior.

*NOTE: It’s important to note that I called the location first to verify the order was legitimate. There was no attempt to modify the price, access restricted information, or in any way cause damage to the site or the business. The only data I modified was the request my own web browser was sending. Stay out of trouble, folks!*
