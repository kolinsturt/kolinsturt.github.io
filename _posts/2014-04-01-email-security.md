---
layout: post
category : lessons
tagline: "Email security tips"
title: Email Security and Privacy
description: Email security and privacy tips
author: Kolin Stürt
tags : [Apple, Email, encryption, iPhone, security, privacy]
---
{% include JB/setup %}

## Email Security and Privacy

There are a lot of privacy related chat apps that offer end-to-end encryption such as [Signal](https://signal.org/). In a [previous article](https://kolinsturt.github.io/lessons/2020/02/01/remote_collaboration) I discuss [Secure Remote collaboration](https://kolinsturt.github.io/lessons/2020/02/01/remote_collaboration) tools you can use. But when it comes to email, it was not designed with security in mind. If you must send email, here's a few tips.

### Getting Started With Basics

First off, you'll want to make sure your accessing email over an encrypted session - SSL. For web based email, make sure all addresses are **https://**, not **http://**. Most security researchers agree that Firefox is a secure browser over the others. There's a good browser plugin for this called [HTTPSEverywhere](https://www.eff.org/Https-everywhere) that attempts to make sure all the pages you browse are using the **https** encrypted versions of the site.

[Privacy Badger](https://privacybadger.org/) and [uBlock Origin](https://github.com/gorhill/uBlock) are good browser plugins to prevent tracking. Adblock plus used to be a very popular plugin, but has since decided to [allow companies to pay](https://www.businessinsider.com/google-microsoft-amazon-taboola-pay-adblock-plus-to-stop-blocking-their-ads-2015-2) to get around their block list, so this one is no longer recommended.

Email desktop applications should also [use SSL](https://riseup.net/en/security/network-security/secure-connections) (ports 587 and 995, not 25 or 110). For mobile devices, instructions to enable SSL encryption are [here](https://support.godaddy.com/help/article/4888/enabling-email-with-ssl-on-your-iphone).

### Encrypting the Content

There are solutions to encrypt your content such as [S/MIME](https://support.microsoft.com/en-us/office/encrypt-messages-by-using-s-mime-in-outlook-on-the-web-878c79fc-7088-4b39-966f-14512658f480) but by far the most secure solution for email is Pretty Good Privacy (PGP), or it's open source equivalent, [GPG](https://gpgtools.org/). The main drawback is that the other person receiving email needs to also have GPG set up as well.

GPG works by exchanging public/private keys between you and the sender; establishing end-to-end encryption. GPG encrypts your email right until the user on the receiving end decrypts it. Other email services may encrypt an email but decrypt it once arrived at their location. There's a great [video](https://www.youtube.com/watch?v=LkRbAFxOu6o) with instructions on setting it up here. You can also check out [written instructions](https://riseup.net/en/security/message-security/openpgp).

There are solutions that automate the encryption steps for you. [Protonmail](https://protonmail.com/) is a great example. It's simple to use but sending to a non Protonmail address is still unencrypted unless you incorporate GPG.

If you want to send encrypted content by email without any changes to your email setup, you can include an attachment that's an encrypted archive. You'll need to disclose the password to the other party by some other means. Make sure it's over a secure channel such as [Signal](https://signal.org/).

Here's an example: if you're both using macOS, you can create an [encrypted disk image](https://support.apple.com/en-ca/guide/disk-utility/dskutl11888/mac). From there add your content, text file, word doc or any files you want. Then send the encrypted disk image to the other person. If they don’t use the same platform as you, [VeraCrypt](https://www.veracrypt.fr/en/Home.html) is the recommended multi-platform encryption program.

NOTE: You should also use this to send files over services such as WeTransfer or YouSendIt. [FireFox Send](https://send.firefox.com/) and [Riseup Share](https://share.riseup.net) are secure alternatives to these services.

As a last resort, you can send a [password protected](https://www.liberiangeek.net/2013/07/password-protect-your-documents-when-using-libreoffice/) Word/Libre/OpenOffice document in lieu of an encrypted archive.

### Anonymizing Your Location

Encryption protects your content but it doesn't protect your location. To do that, access your email using it's web-based version over [Tor Browser](https://www.torproject.org/projects/torbrowser.html). Tor Browser hides your true location.

Desktop email apps like Outlook or Apple Mail may send other information about who you are for analytics and debugging purposes. If you want to use an app instead of web-based, use products that are open-sourced. With open-sourced software, security researchers can review the code to make sure it doesn't contain spyware. [Thunderbird](https://www.mozilla.org/en-US/thunderbird/) is a good open sourced email client.

Tor Browser does not protect your other apps that connect to the internet. There is a plugin for ThunderBird called [TorBirdy](https://addons.mozilla.org/En-us/thunderbird/addon/torbirdy/). It routes ThunderBird connection through the Tor network so that the receiver will not know your location. Using TorBirdy with Thunderbird is a good solution. 

### Forensic Email Evidence 

You've encrypted your email and anonymized your location. However, your computer may contain cache or historic data of your emails. 

A workaround is to use a bootable live OS that wipes it’s memory afterward. You'd use the OS while working with and sending sensitive information. [Tails](https://tails.boum.org/) is a great solution. It works with TorBrowser out of the box. 

### Working on Mobile 

These tools are all good if you are on a desktop computer but on a mobile device these tools are less available. Apple released [instructions to enable S/MIME](https://support.apple.com/en-ca/HT202345).

A paid enterprise solution is [Blackphone](https://www.silentcircle.com/products-and-solutions/blackphone2/) or [iPGMail](https://apps.apple.com/us/app/ipgmail/id430780873) for GPG. 

But the most popular secure app that is free is still [Signal](https://signal.org/).

#### Anonymizing Your Mobile Location

First, keep in mind that the phone company knows where you are. You can buy a burner phone in cash but remember that the cell towers can triangulate where you are. An extreme solution is to have a second device such as an iPod touch for sensitive correspondence. That way it doesn't ever connect to a cell tower.

In either case, you'll want to use a VPN on the device. Instructions for setting this up are [here](https://www.vpnbook.com/howto/setup-openvpn-on-ipad). A free VPN is [VPNBook](https://www.vpnbook.com/). However, paid VPNs like [LiquidVPN](https://www.liquidvpn.com/) and [Pia VPN](https://www.privateinternetaccess.com/) make it clear that they do not keep any traffic logs.

As a last resort, you can use a proxy such as one found on [proxylist.hidemyass.com](http://proxylist.hidemyass.com). Instructions on setting up a proxy are [here](http://www.amsys.co.uk/2012/blog/how-to-setup-proxy-servers-in-ios/). This is not an ideal solution compared to a VPN. 

With free proxies, security is even more placed at the mercy of the proxy - some proxies keep detailed logs and could be a honeypot. Additionally, iOS apps that use lower level socket communications in place of standard communication frameworks bypass the proxy. Some example apps that bypass proxy settings include Clash of Clans, KIK Messenger, Opera Mini, Pinger, Spotify Premium, and Tango.

For the technically inclined, a robust option is to set up your router to use a VPN and then all your connected devices are automatically routed through it (minus cell connections).

## Conclusion

Last but not least, your computer can capture email content if it has spyware or malware installed. You'll want to install a virus scanner to prevent this. ClamAV is free and open-sourced but requires you to be tech savvy to install it.

These suggestions are for the casual user who wants to add layers of privacy to their communications. If you're working on very sensitive content, check out the [Privacy For Investigators and Whistleblowers](https://kolinsturt.github.io/lessons/2018/06/01/privacy_activists_investigators) article.