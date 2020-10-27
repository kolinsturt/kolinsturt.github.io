---
layout: post
category : lessons
tagline: "Beginning iOS Forensics"
title: iOS Forensics Part 1
author: Kolin Stürt
description: Tools to aid in iOS forensics and jailbreaks.
tags : [iOS, jailbreak, forensics, investigation, root, bypass lock screen]
---
{% include JB/setup %}

# Beginning iOS Forensics

This is a brief article that introduces the concepts of forensics on iOS. It’s not meant to be a full guide but rather to present the general idea and overall process that you can take away and apply to your unique situation.

You may get lucky when presented with an unsecured device where you have full access. But the most important practice for any device is to copy and work with the data without altering the original device in any way. Evidence that is tampered with will be inadmissible in court.

Beyond copying, backing up and preserving the current state of the device, the next main challenge in forensics recovery is dealing with a secured device. Almost all solutions involve jailbreaking the device to gain access. There is a tradeoff between acquiring access and altering part of the device that secures it. 


# Jailbreaking

Jailbreaking allows you to get around the sandbox and kernel patch protection. Most importantly, it gives you root access to the file system as well as the opportunity to enable SSH on the device.

There are many tools to root a device, such as:

* [doubleh3lix](https://doubleh3lix.tihmstar.net/)
* [liberios](http://newosxbook.com/liberios/)
* [electra](https://coolstar.org/electra/)

Rooting a device usually involves exploiting a security vulnerability on the OS which would circumvent code signing and other security measures. Code signing prevents you from modifying or executing custom code on the device. An IPA image, for example, is signed by a private key and the main challenge is getting unsigned binaries to run. [Cydia Impactor](https://www.cydiaimpactor.com/) is used to get around these requirements to install jailbreaks. 

Instead of trying to escalate privileges to get root, newer iOS 12 techniques involve a rootless jailbreak. This doesn't need to grant access to the root (/) so it's much safer from a forensics standpoint. There's less modifications to the evidence.

Here's some more resources:
* [Rootless Jailbreak JB3](https://github.com/jakeajames/rootlessJB3)
* [Apple’s iOS Kernel Patch Protection (KPP) Explained](https://yalujailbreak.net/kernel-patch-protection/)
* [Rootless Vs Full Root Jailbreak](https://www.redmondpie.com/rootless-vs-full-root-jailbreak-on-ios-12-whats-the-difference/amp/)


# Extracting Data

A physical acquisition is a bit-by-bit dump of the storage. A logical acquisition on the other hand does not do a bit-by-bit copy. It’s similar to the process of copying a file or folder from one location to another. A logical acquisition is usually the technique used if you can get the lockdown file (a pairing record) from the user's computer. This doesn't work if the device has been started for the first time without the user unlocking it at least once. (Why it's a good idea to power down your device in high-risk situations.)

Starting in iOS 11.3, locked devices need to have had a lightning connector or USB accessory used while unlocked at least once a week., otherwise the connection becomes unusable. This feature is called USB Restricted Mode. It means that you must act promptly when given a secured device otherwise the lockdown records will expire after a week. On 11.4.1 it gets worse. The timing window was changed to one hour. This change is in response to [Grayshift’s GrayKey](https://graykey.grayshift.com/), which has been the leading device to break into locked devices via the USB connector.

However, it seems there's some workarounds. Connecting certain accessories such as display adaptors or Apple's Lightning to USB 3 Camera Adapter changes this window to a week.

Apple introduced the Secure Enclave starting with the iPhone 5s devices. Prior to these devices, you can do a physical acquisition of the user’s encrypted data partition as long as you could extract the encryption key. For devices with the Secure Enclave, you'll need to perform the higher level system imaging acquisition.

* Once the phone is jailbroken, you can use a simple too like [iExplorer](https://macroplant.com/iexplorer) on the device.
* You can SSH into the device:
  ssh root@10.0.0.198
* You can use a tool like [cyberduck](https://cyberduck.io/) to copy files to your computer. 
* A live data imaging tool that may be helpful is dd. It is located at /system/bin.

Another method is to look at an iTunes backup if available. Both [iPhone Backup Extractor](https://www.iphonebackupextractor.com/) and [iExplorer](https://macroplant.com/iexplorer/export-iphone-notes-call-history-calendar-safari) allow you to extract contents from an iTunes backup. Note that the device UDID is the name of the device’s backup file. In newer A12 chip devices like the X series, the UDID is much longer.

The locations for backups are as follows:

* macOS ~/Library/Application Support/MobileSync/Backup/
* Windows C:Users<username>AppDataRoamingApple ComputerMobileSyncBackup
* Windows via the Microsoft Store: C:Users<username>AppleMobileSyncBackup

If the backup has been encrypted, you'll need to run a password cracking utility against the manifest.plist file inside the backup.

# Location of User Data

Once you have access to the device, you can begin with important areas of data:

* iTunes apps are located in **private/var/mobile/Applications**.
* Each app folder have **documents**, **temp** and **library** subfolders.

* Photos are located in **private/var/mobile/media/DCIM**.
* SMS: **/private/var/mobile/Library/SMS**
* Call History: **/private/var/Library/CallHistory**
* GPS Locations: **private/var/Library/Caches/locationd/consolidated.db**
* The keyboard cache contains new words the user has typed. This can help determine content of messages. The cache is located at **/private/var/mobile/Library/Keyboard**.
* Keychain items are located at **/private/var/Keychains**. You can use [Keychain-Dumper](https://github.com/ptoomey3/Keychain-Dumper) to assist in extracting contents.
* Safari Cache: **/private/var/mobile/Library/Caches/Safari**
* Cookies: **/private/var/mobile/LibrarySafari/cookies.binarycookies**
* Notes: **/private/var/mobile/Library/Notes**
* Address Book: **/private/var/mobile/Library/AddressBook**

It would be hard to keep track of all the 3rd party apps, so here's some notes about a few:
## WhatsApp
On iOS, if you have access to a phone backup, the messages are not encrypted! This is likely because the developers assumed the iOS platform was more secure, which is a mistake. Also, see this [interesting iOS article](https://www.zdziarski.com/blog/?p=6143) about WhatsApp artifacts. On Android, the messages are encrypted. See [Decrypting WhatsApp Backups](https://blog.elcomsoft.com/2018/12/a-new-method-for-decrypting-whatsapp-backups/) for more information.

## WeChat
See the [Decrypt WeChat Database article](https://articles.forensicfocus.com/2014/10/01/decrypt-wechat-enmicromsgdb-database/) for more information.

* Most native apps like Calendar, Messages and Address Book use SQLite to store their data.
* SQLite is a relational database management system embedded into the end program. It's like SQL but is not a client–server database engine.
* You can analyze SQLite files by getting the free [sqlitebrowser](https://sqlitebrowser.org/).

# Recovering Deleted Data

Most of the above data is located inside sqlite files. For sqlite, there are unallocated blocks and free blocks. Like a typical file system, when you remove an entity it's simply marked as free but not overwritten immediately. That means you can use a Hex viewer that also shows ASCII to search for keywords that may still be present. File carving is the process of finding and extracting data from storage where you do not have all the information about the file.

There's some valuable information regarding sqlite file carving [here](https://forensicsfromthesausagefactory.blogspot.com/2011/04/carving-sqlite-databases-from.html).

[Scalpel](https://github.com/sleuthkit/scalpel) is an open-sourced data carving tool.

A good tool for viewing an sqlite database is [sqliteviewer](http://www.oxygen-forensic.com/en/features/sqliteviewer/).

Besides message databases, another important thing to look for are pictures. For example, in the JPEG format the first two bytes and last two bytes are always **FF D8** and **FF D9**.

  
# Where to Read More

So that's a brief overview of the forensics process. For more information check out the following:

* [JTAG and Chip-off methods](https://hitcon.org/2016/CMT/slide/day1-r1-d-1.pdf)
* [Autopsy - The open source digital forensics platform](https://www.autopsy.com/)
* [Santoku Linux - Mobile and malware forensics](https://santoku-linux.com/)
           
Sometimes apps use obfuscated or eccentric ways of storing the user's data so the focus shifts to understanding the particular app and how it works internally. During the forensic process, you can get a lot of intel by analyzing the app itself. Check out the [Hacking iOS Apps](https://kolinsturt.github.io/lessons/2015/01/01/iOS_hacking) article next to learn how to do that.

