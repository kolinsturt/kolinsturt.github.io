---
layout: post
category : lessons
tagline: "Privacy for Activists and Investigators"
description: Tools to help with privacy for activists.
tags : [Signal App, VeraCrypt, TorBrowser, Tails OS, HTTPSEverywhere, Riseup.net, Autistici, Whisper Systems]
---
{% include JB/setup %}

## Privacy for Activists and Investigators

This guide will discuss how to communicate, coordinate, and expose information as securely as
possible. You'll read about computer security first, followed by mobile security.

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
are doing. A good VPN option is [Riseup.net](https://riseup.net/en/vpn), or [LiquidVPN](https://www.liquidvpn.com/).

Using a VPN is good for when you need to send and receive sensitive information at work or while
connected through your internet provider. VPN would be good to use at an airport WIFI where the
traffic is regularly monitored. It's also a good solution to thwart government censorship.
The drawback to VPNs are that the VPN provider can see your traffic. Some VPN providers don't keep
any records or logs whatsoever and are set up in countries that have the strongest privacy laws.
[Riseup.net](https://riseup.net/en/vpn) is one such company and their VPN service is free. A good paid solution is [LiquidVPN](https://www.liquidvpn.com/). You should choose a VPN provider depending on the sensitivity of the work
you're doing. If you want to hide traffic from whomever operates the WIFI hotspot you're connecting
to, LiquidVPN is fine. While both companies state they do not keep logs, if you're worried about law enforcement obtaining a warrant from the VPN company and checking logs, Riseup would be a good choice. [Autistici](https://www.autistici.org) is an
alternative to the Riseup.net platform.

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
VeraCrypt also offers a feature to create a hidden volume, which is good in situations where you're
forced to reveal your passcode under duress. You could reveal the encrypted archive that contains
“normal” information without revealing the hidden archive - https://veracrypt.codeplex.com/wikipage?title=Hidden%20Volume

## Mobile Security
### Communications
Texting has become the most prominent form of communication. Regular cell phone texts are not
secure. Here are the secure app alternatives:
#### [Signal](https://whispersystems.org)
Signal is an encrypted chat app that is available for Android, iPhone, and
desktop computers. Messages get encrypted so that only the sender and receiver can read them. (called
end-to-end encryption). You can also choose to have messages disappear after a specific time once
viewed. No one yet has broken the encryption of this app and the full code for the app is open for
public review. The security community recommends Signal because it offers the strongest protection.
The downside is that it's not as popular as some other chat platforms such as WhatsApp. WhatsApp
uses the Signal technology, but WhatsApp is a for-profit company and the app is proprietary. That
means that you have to trust that they haven't altered the software to include backdoors that would
allow third parties to spy on the communications (https://signal.org/blog/there-is-no-whatsapp-backdoor). I've worked on the Signal app so feel free to message me questions. 

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
* Telegram is heavily used by Hong Kong protesters to protect monitoring by authorities. It had a vulnerability that let authorities match protesters' real names to their profiles in anonymous groups. It was fixed in a software update so you should use the most recent version of Telegram - https://coconuts.co/hongkong/news/telegram-closes-privacy-loophole-to-protect-hong-kong-protesters/.

### Storage
Securing your communications is great, but forensic traces still exist on your mobile device. Here are
some tips to secure your data at rest:
1. On Android, enable [Full Disk Encryption](https://www.howtogeek.com/141953/how-to-encrypt-your-android-phone-and-why-you-might-want-to)
2. Update your operating system to the latest version. Hackers and security researchers find weaknesses
in mobile operating systems. Good researchers report the vulnerabilities and they're fixed during
software updates. (eg https://support.apple.com/en-ca/HT207923). Not all security researches are
ethical, so undisclosed vulnerabilities still exist. (We saw this with the WikiLeaks “Vault 7” release).
Because of the researchers, newer versions of the operating system are more hardened than their
previous versions. Forensics software used by law enforcement - [ElcomSoft](https://www.elcomsoft.com)
and [Encase](https://www.guidancesoftware.com/encase-forensic) can not break into the newer
software, provided there is a strong passcode on the device.

### Email and Social Media
Email, as well as most social network messaging were not designed with security in mind. It's almost
impossible to be anonymous if you're using social media. Your posts, likes, interests, and who you
communicate with get linked together to form a bigger picture. Consider using a separate account for
sensitive activity. If you must use email, there's a way to encrypt it by setting up [GPG](https://gpgtools.org), but it requires work for both communicating parties to set it up. [Riseup.net](https://riseup.net) offers
solid encrypted storage, but remember than any emails you send out to individuals are in the clear and
saved on multiple servers. It's better to stick to secure chat apps such as Signal and Telegram for
sensitive communications.

### Crossing The Border
If asked to unlock your device, you have to think how to respond. You can state that you do not
consent, knowing there is a chance security will hold you for a long time and force you to unlock your
device. There are several options available. The best is to prepare before you travel. If you have a cloud
account, you can backup your device and then reset it (erase it to factory settings) before traveling.
After you've crossed the border, you can restore the device and data back from your cloud account. A
safer option is to leave your devices behind and carry a loaner or cheap throw away, pay-as-you-go
phone. If you don't trust cloud accounts, you can backup your device to an external hard-drive at home
and then restore the device upon returning. If you go the cloud route, make sure you don't carry any
information revealing that you have an account. Security can demand your passwords to Google,
iCloud, and social media accounts. Make sure you don't carry any evidence of their existence. One
option is to set your social media accounts to hidden or private while you cross the border.

#### Law Enforcement
Modern forensics software used by law enforcement can extract information from a powered on, and
locked device. When a device gets turned off, the master keys are gone until you power on the device
and unlock it. You should power down your device in areas where security can confiscate it. This will
render their forensic equipment useless. Newer iPhones have a feature where you can press the lock
button 5 times in a row to enable duress mode. During duress mode, only your passcode can unlock the
device. Unlocking the phone via fingerprint or facial recognition will be unsuccessful. In a situation
where you think there is chance of an arrest, or during a medical emergency, you should take advantage
of this. That's because when you're arrested or passed out, law enforcement can hold your finger or face
up to the phone to unlock it. In Canada, a police officer is not allowed to unlock your phone during a
roadside stop. If arrested and handcuffed, an officer can take your phone and use your finger or face to
unlock the device, so it's best to use a strong passcode in this case.


## Protection From Infiltrators

Often infiltrators are there to try to get you to break the law. An obvious sign of an infiltrator will be when someone is urging you to be destructive or violent. They may try to joke with you about an illegal activity in hopes of capturing incriminating evidence. Even if you think you're among trusted friends, it’s best not to joke about these things because you don’t know if someone is recording you. (It’s best to assume you are always being recorded). I’ve seen hypothetical posts on social media that are emotional and written in haste. For example “burn them down to the ground!”. While you might not mean it, it may be used against you in the future. Anything you put on the internet, from a comment to a photo that you liked, is very hard to redact. Someone can put all the little pieces together to form a bigger picture and turn it against you later.

You can still be honest about an upcoming event without divulging all the details until it’s necessary. To quote Robert Greene, “Always say less than necessary”. That why, you’re not tipping off an infiltrator, nor admitting specific details.

Keep in mind that when you create pages or posts online and only intend of sending a few key people the link, others will still immediately know you’ve posted about them with technologies like [Google Alerts](https://www.google.ca/alerts). Instead, post it privately such as a document on [Riseup's Share](https://share.riseup.net).



## Street Police

You are not required to talk or answer questions to law enforcement when approached on the street. You have a right to remain silent, but it's illegal to lie or provide fake documents to a police officer. Still, it’s always best to be polite. You may ask for their badge number and what force they are from. The opposite isn't true unfortunately. A police officer can lie to you or mislead you to get you talking, for example. You are not required to show ID or provide your name or address. You're required to give your name if a police officer is ticketing you. If you’re ticketed for a bicycle offense, you're required to provide both your name and address.

In regards to not having to show ID, there are exceptions when you’re performing a task that requires a license. A good example is when driving a car, or drinking alcohol in an establishment.

If you’re stopped while driving, the police are not allowed to search your car unless they have reasonable and probable grounds that there are illegal items in the car (such as an item that an officer sees by looking through the window).

In regards to your home, you do not have to answer the door or let police into your place, unless they have a search or arrest warrant. Police can enter if there are signs of an immediate emergency in which they need to attend to.

### Arrests and Detainments

To figure out if you are under arrest or detained, ask if you are under arrest, and why. If not, ask if you are free to leave, and if not, why.

If you are detained, you have the right to speak to a lawyer, and you do not have to answer any questions. They can do a pat-down for their own safety, but not a search.

In all these situations, you can state that you do not consent to a search. If an officer searches you anyway, don't resist, but do continue to say "I do not consent to this search."

If you are arrested, an officer can search you. You must be provided promptly with the reason for the arrest, and police must advise you of the right to speak to a lawyer (in private, and includes multiple phones calls in order to reach a lawyer). You have the right to remain silent, so you do not have to answer any of the police officer's questions.

It's helpful to memorize the number of a lawyer or important people in your life that you may need to call.

Police can only attempt to search your phone on arrest, however, you do not have to give them your password to unlock the device. They may use forensic tools on it anyway but it’s not always successful. Currently the iOS 12 update (USB Restricted Mode) had stopped much of the police force's ability to unlock the phone with their [GrayKey](https://graykey.grayshift.com/) devices. It’s a cat and mouse game. A new workaround is available and it’s only a matter of time before all the forces get the update, but by then Apple may have patched it again.

In regards to the private messaging app Signal, you can add a lock screen on app open. If someone obtains your phone unlocked or does a data dump of the device, the signal database will still be encrypted. (I helped work on this part of the Signal App so feel free to ask me any questions about it's implementation)

In the event any of your electronic devices are taken from you, login to your social media account dashboards as soon as possible and sign out of all connected devices. Then change the passwords to all your accounts so they can not still be accessed.

For any law enforcement interaction, always remain calm and polite. Record and document everything you can such as badge numbers, car numbers, and license plates.

And lastly, this section is for generalized information. It should not be used in place of legal advice.

## Where to Go From Here
As an investigator or whistle blower, it's important to have a secure working environment. You'll collect
a large amount of information during your research, and it's important to contact sources and
informants in a safe way. Most of your data gets stored and transmitted in a digital format. Ensure that
both your data at rest and data in transit are as safe as possible at all times.
