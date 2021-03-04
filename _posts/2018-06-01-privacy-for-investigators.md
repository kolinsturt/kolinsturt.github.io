---
layout: post
category : lessons
tagline: "Privacy for Investigators"
title: Privacy for Investigators
description: Privacy tools for investigators and researchers
tags : [Signal App, VeraCrypt, TorBrowser, Tails OS]
redirect_from:
  - /lessons/2018/06/01/privacy_activists_investigators
  - /lessons/2018/06/01/privacy-activists-investigators
---
{% include JB/setup %}

## Privacy for Investigators

This guide will discuss how to communicate and coordinate information as securely as
possible for investigators and researchers. You'll read about computer security first, followed by mobile security.

## Laptops and Desktops

### Internet Browsing
#### [TorBrowser](https://www.torproject.org/projects/torbrowser.html.en)
TorBrowser acts like the Firefox internet browser but hides your location from sites you're visiting.
It does this by concealing your IP address through a series of virtual tunnels. Companies usually keep
track of their visitors for marketing and security purposes. But if you visit example.com through
TorBrowser, that company will not have records of who or where you are. This is beneficial if you're
investigating a company and don't want them to have any trace of your investigation.
TorBrowser doesn't hide (encrypt) your communications from eavesdroppers. For example, if you're
using a public WIFI hotspot at Starbucks, it's possible for someone else on the same WIFI spot to
snoop on your activity. To conceal your browsing traffic from eavesdroppers:
#### Use HTTPS when Visiting Websites
Make sure the site you're visiting starts with https:// and not http://. Sites that start with https encrypt
your traffic. This is important in cases where you're submitting or receiving information that you don't
want intercepted. There's a plugin for FireFox called [HTTPSEverywhere](https://www.eff.org/https-everywhere) that switches the sites you visit from http to https. Note that some websites don't offer
https versions.

#### VPN
Even when you're using an https version of a site, the internet service provider you're using (whether at
work or home) can intercept your communications (called an interception proxy). To safeguard against
this, use a VPN (Virtual Private Network). VPNs will conceal (encrypt) and forward all traffic from
you over to the VPN company. Anyone trying to snoop on your traffic will not be able to see what you
are doing. A good VPN option is [ProtonVPN.com](https://protonvpn.com).

Using a VPN is good for when you need to send and receive sensitive information at work or while
connected through your internet provider. VPN would be good to use at an airport WIFI where the
traffic is regularly monitored. It's also a good solution to thwart government censorship.
The drawback to VPNs are that the VPN provider can see your traffic. Some VPN providers don't keep
any records or logs whatsoever and are set up in countries that have the strongest privacy laws. You should choose a VPN provider depending on the sensitivity of the work you're doing. If you want to hide traffic from whomever operates the WIFI hotspot you're connecting
to, LiquidVPN is fine. While both companies state they do not keep logs, if you're worried about law enforcement obtaining a warrant from the VPN company and checking logs, Riseup would be a good choice.

#### [TailsOS](https://tails.boum.org)
The above techniques protect your information in transit, but they don't protect the data saved on your
computer. A trail of forensic evidence gets left behind while you work on your computer. If what you're
doing is so sensitive that law enforcement will confiscate your computer, TailsOS is the solution.
TailsOS is an “amnesic” operating system for your computer that doesn't leave traces behind. It starts
up from a USB stick or DVD and will never store anything to your hard drive. The memory gets erased
once you shutdown the system. It's designed for sensitive situations where a forensics search of your
computer won’t reveal anything that you've done using the TailsOS.
Storage

### Encrypting Your Data
While it's great to cover your tracks using the methods above, you'll need to save your work
somewhere. You should encrypt your investigative files so that if they're ever found, it would be
infeasible to read them without knowing your password (don't choose an easy to guess password). The
best encryption software is [VeraCrypt](https://veracrypt.codeplex.com). This software is open sourced.
Open Sourced means that the full code for the program is open to the public so that security researches
can audit it to make sure there are no weaknesses or back doors. VeraCrypt creates encrypted folders of
any size you want, and you can send sensitive content over the internet via VeraCrypt provided both
parties know the passcode. (This is similar to sending someone a password protected zip-file).
VeraCrypt also offers a feature to create a [hidden volume](https://veracrypt.codeplex.com/wikipage?title=Hidden%20Volume), which is good in situations where you're forced to reveal your passcode under duress. You could reveal the encrypted archive that contains “normal” information without revealing the hidden archive.

## Mobile Security
### Communications
Texting has become the most prominent form of communication. Regular cell phone texts are not
secure. Here are the secure app alternatives:
#### [Signal](https://signal.org)
Signal is an encrypted chat app that is available for Android, iPhone, and
desktop computers. Messages get encrypted so that only the sender and receiver can read them. (called
end-to-end encryption). You can also choose to have messages disappear after a specific time once
viewed. No one yet has broken the encryption of this app and the full code for the app is open for
public review. The security community recommends Signal because it offers the strongest protection.
It's not as popular as WhatsApp. While WhatsApp uses the Signal technology, it's not recommended as a secure choice. That's because WhatsApp is a for-profit company, owned by Facebook and the app is proprietary. That means that you have to trust that they haven't altered the software to include backdoors that would
allow third parties to spy on the communications ([https://signal.org/blog/there-is-no-whatsapp-backdoor](https://signal.org/blog/there-is-no-whatsapp-backdoor)). 

#### [Telegram](https://telegram.org)
Telegram is very popular. It allows you to redact a message you've sent,
even off the other person's device. This is a false sense of security because there's no way of
determining if the other user has already seen the message. It's possible to put the device into airplane
mode and read a message before it's redacted. Telegram also provides “screenshot security”. It disables
taking screenshots on Android, and on iPhone lets the other party know that a user has taken a
screenshot. This is also a false sense of security. A user can take a picture of the conversation using
another phone or camera. Telegram is #2 to Signal for that and the following reasons:
* Unlike Signal, it doesn't encrypt conversations by default; each user must start a “secret chat”.
* In 2015, researches found theoretical weaknesses in the encryption algorithm. Signal uses time-tested
open standards (ECDSA) for the encryption that contain no weaknesses discovered to date.
* Part of their source code is open to review while the rest is proprietary. Signal's code is open-sourced.
(Code that is open-sourced is available for the public to review and audit). Besides Signal, Telegram is
still more secure in comparison to any other chat app.
* Telegram is heavily used by Hong Kong protesters to protect monitoring by authorities. It had a vulnerability that let authorities match protesters' real names to their profiles in anonymous groups. It was fixed in a software update so you should use the most recent version of Telegram - [https://coconuts.co/hongkong/news/telegram-closes-privacy-loophole-to-protect-hong-kong-protesters/](https://coconuts.co/hongkong/news/telegram-closes-privacy-loophole-to-protect-hong-kong-protesters/).

### Storage
Securing your communications is great, but forensic traces still exist on your mobile device. Here are
some tips to secure your data at rest:
1. On Android, enable [Full Disk Encryption](https://www.howtogeek.com/141953/how-to-encrypt-your-android-phone-and-why-you-might-want-to)
2. Update your operating system to the latest version. Hackers and security researchers find weaknesses
in mobile operating systems. Good researchers report the vulnerabilities and they're fixed during
software updates. (eg [https://support.apple.com/en-ca/HT207923](https://support.apple.com/en-ca/HT207923)). Not all security researches are
ethical, so undisclosed vulnerabilities still exist. (We saw this with the WikiLeaks “Vault 7” release).
Because of the researchers, newer versions of the operating system are more hardened than their
previous versions. Forensics software used by law enforcement - [ElcomSoft](https://www.elcomsoft.com)
and [Encase](https://www.guidancesoftware.com/encase-forensic) can not break into the newer software, provided there is a strong passcode on the device.

### Email and Social Media
Email, as well as most social network messaging were not designed with security in mind. It's almost
impossible to be anonymous if you're using social media. Your posts, likes, interests, and who you
communicate with get linked together to form a bigger picture. Consider using a separate account for
sensitive activity. If you must use email, there's a way to encrypt it by setting up [GPG](https://gpgtools.org), but it requires work for both communicating parties to set it up. Activists involved with Black Lives Matter protests heavily use [Tutanota](https://tutanota.com/) and [ProtonMail](https://protonmail.com) offer
solid encrypted storage, but assume emails you send out to individuals are in the clear and
saved in multiple places. It's better to stick to secure chat apps such as Signal and Telegram for
sensitive communications.

## Where to Go From Here
As an investigator, it's important to have a secure working environment. You'll collect
a large amount of information during your research, and it's important to contact sources and
informants in a safe way. Most of your data gets stored and transmitted in a digital format. Ensure that
both your data at rest and data in transit are as safe as possible at all times.
