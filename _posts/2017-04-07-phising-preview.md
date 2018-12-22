---
title: Phishing in the age of preview
author: Kobi
comments: true
---
Link previews are convenient. Implemented inside messaging platforms, they allow you to see the summary of a link a friend sent you without actually opening it, giving you the ability to decide whether it's important or interesting enough to open it in a browser.

 Consider the following situation: Your WhatsApp account has been hacked. The hacker goes through your contacts list and message history, finding out who are your the people you're conversing with the most. Then, the hacker prepares a malicious web-site - the web-site installs some malware that steals your bank account password once you go into it. The hacker even makes the web-site have a nice preview about a subject you're discussing with your friend often, so your friend would think it's a legitimate web-site. The friend, not knowing he's talking with a hacker rather than his good friend, opens the link and gets infected!

This may sound unrealistic - a messaging account getting hacked, but it happened to a colleague of mine in [QED-it](http://qed-it.com). His Skype account got hacked and started distributing suspicious links to some of his contacts. Apparently, it was quite successful, as some of these contacts got hacked as well and sent him suspicious links back.

So while it's a multi-step phishing attack, once you get hold of a social media or messaging account, if you can construct believable links, it should be pretty easy to infect your contacts.

#Wait, what?

You must think - "well, this guy is crazy, it's not that easy creating these links". Is it really that easy to construct believable previews? Apparently yes! All you have to do is implement the [Open Graph Protocol](http://ogp.me/) on your web-page. This is a description of the Open Graph protocol, in their words:
> The Open Graph protocol enables any web page to become a rich object in a social graph. For instance, this is used on Facebook to allow any web page to have the same functionality as any other object on Facebook.

Many of the messaging platforms support it - an incomplete list includes Facebook, WhatsApp and Telegram.

In order to have a preview for your web-site, you just need to add this prefix to the `html` tag and these properties to your `head` section:
```
<html prefix="og: http://ogp.me/ns#">
    <head> 
        <meta property="og:title" content="">
        <meta property="og:site_name" content="">
        <meta property="og:image" content="">
        <meta property="og:url" content="">
        <meta property="og:description" content="">
    </head>
</html>
```
That's it. The hacker can now insert whatever code he wants, even a redirect to another web-site.

Another component which may be useful in order to mask the fact it's a malicious URL would be a URL shortener such as [bit.ly](http://bit.ly).

#Meh, that's a bit of work
Well, then you can just use [my tool](http://roll.kobigurk.com/generator.html) to generate an auto-redirecting malicious URL shortened by bit.ly.

![TPaaS](/content/images/2017/04/Untitled.png)

Now, that's really easy...

This, [http://bit.ly/2oCoEeY](http://bit.ly/2oCoEeY) , is a link that has a preview exactly like a real BBC article, but actually rick-rolls you (advantage if you like Rick Astley music videos!). This is its preview as it's rendered on Telegram:
![TelegramExample](/content/images/2017/04/Untitled-1.png)

#What can we do?
Good question. It's always an interactive game - put a barrier, hackers get through and so on. But, in my feeling, there's a lot of room for improvement regarding these previews.

Facebook, for example, does a better job than the rest:
1. If a URL is given in the `og:url` tag, that one is eventually used to retrieve the others.
2. If a URL is not given or invalid, then the actual URL is displayed under the post.

In Telegram and WhatsApp, this is not the case - you completely control the content that is shown to the user.

So, to begin with, sites can show the actual URL being used to retrieve the tags.

A different direction I've been considering is similar to the HTTPS user experience existing in most major browsers:
1. If the URL is not authenticated (HTTP), show a message that says "this preview is not secure".
2. If the URL is from an HTTPS link, show the Common Name in the certificate.

These details alone would make the situation much better.

#To summarize...
Link previews are nice and fun, but may also be used as a convenient tool to improve phishing campaigns considerably. I believe there are solutions to make it better and some platforms are realizing this and taking steps, but there's more work to do. 

Hopefully we can combine the easiness and convenience with security!
