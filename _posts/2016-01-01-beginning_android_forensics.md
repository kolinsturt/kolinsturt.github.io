---
layout: post
category : lessons
tagline: "Beginning Android Forensics"
description: Tools to aid in Android forensics - Kolin Stürt.
tags : [Android, aapt, forensics, investigation, adb, root, bypass lock screen, unlocking bootloader]
---
{% include JB/setup %}

# Beginning Android Forensics

This is a brief article that introduces the concepts of forensics on Android. It’s not meant to be a full guide but rather to present the general idea and overall process that you can take away and apply to your unique situation.

You may get lucky when presented with an unsecured device where you have full access. But the most important practice for any device is to copy and work with the data without altering the original device in any way. Evidence that is tampered with will be inadmissible in court.

Beyond copying, backing up and preserving the current state of the device, the next main challenge in forensics recovery is dealing with a secured device. Almost all solutions involve rooting the device to gain access. There is a tradeoff between acquiring access and altering part of the device that secures it. 


# Rooting and Unlocking the Bootloader

There are many tools to root a device, such as [OneClickRoot](https://www.oneclickroot.com/), [KingoRoot](https://www.kingoapp.com/) and [SuperUserDownload](https://superuserdownload.com/).

Rooting a device usually involves flashing a partition on the device, such as a custom recovery image. Some examples are <https://twrp.me/about/> or [ClockworkMod Recovery](https://www.clockworkmod.com/). 

These tools work unless the boot loader is locked. A locked boot loader prevents you from modifying the firmware. Usually the image is signed by a private key so that unsigned code can not be flashed onto the device. There are OEM bootloader unlock commands, but they perform a wipe of the device.

In order to perform a root with a locked bootloader, a security vulnerability in the OS will need to be exploited. This is very similar to iOS where most of the jailbreaks are done via a known exploit. Often you can find help at [XDA-Developers](https://www.xda-developers.com/).

Some examples of exploits are:
* [g2root-kmod](https://github.com/tmzt/g2root-kmod)
* [mempodroid](https://github.com/saurik/mempodroid)
* [dirtycow](https://dirtycow.ninja/)
* [gingerbreak](https://getandroidstuff.com/download-gingerbreak-app-easy-rooting-gingerbread-android-user/
)

For an updated list see:
* [Android-Exploits](https://github.com/sundaysec/Android-Exploits)
* [AndroidVulnerabilities](https://androidvulnerabilities.org)

# Extracting Data

A physical acquisition is a bit-by-bit dump of the storage. A logical acquisition on the other hand does not do a bit-by-bit copy. It’s similar to the process of copying a file or folder from one location to another.

The most basic way to collect data is to use the [adb backup](https://developer.android.com/studio/command-line/adb) command. In order to enable the process, you’ll need to enable [USB debugging](https://developer.android.com/studio/debug/dev-options) on the device. You can also use the **adb pull** command to pull individual files or directories.

A live data imaging tool that may be helpful is *dd*. It is located at **/system/bin**. To extract the current memory state of the device, check out [LiME](https://github.com/504ensicsLabs/LiME).

# Bypassing the Lock Screen

A pattern, pin, or smart lock such as trusted face is likely enabled on the device.

* Pattern locks are located at **/data/system/gesture.key**
* Pin and Passwords are hashed at **/data/system/password.key**
* Those hashes are salted, and the salts are located at **/data/system/locksettings.db**

Tools such as [andriller](https://www.andriller.com/) and [androidpatternlock](https://github.com/sch3m4/androidpatternlock) perform cracking on these files.

While you don’t want to alter evidence, on some devices the lock screen can be bypassed by deleting the files.

[LiME](https://github.com/504ensicsLabs/LiME) could also be used to extract passwords and keys from memory.

# Location of User Data

Once you have access to the device, you can begin with important areas of data:

* All apps store user data in the **/data/data directory**.
* List of apps installed are at **/data/system/packages.list**.
* **/data/system/package-usage.list** shows the last time apps were used.
* Wifi connection information such as a list of access points is located at **/data/misc/wifi/wpa_supplicant.conf**.

Here's location info about common apps. The sub-points are the names of important tables in the database:

### Contacts
* com.android.providers.contacts
* /databases/contacts2.db 
  * calls – call history
  * contacts
  * deleted_contacts
  * data – contains other info such as email address

### Phone
* com.android.providers.telephony
* /databases/mmssms.db
* /databases/telephony.db

### GMail
* com.google.android.gm
* /databases/mailstore.\<user\>@gmail.com.db

### Chrome
* com.android.chrome
* /app_chrome/Default
  * History
  * Cookies
  * Login Data

### Google Maps
* com.google.android.apps.maps
* /databases/gmm_myplaces.db - locations saved
* /databases/gmm_storage.db - searches and locations navigated

### Facebook messenger
* package is com.facebook.orca
* /cache/
  * audio - audio messages
  * fb_temp - sent images and videos
* /databases/prefs_db - Logged in account info
* /databases/threads_db2
  * messages - each message sent and received

### Skype 
* package is com.skype.raider
* /files/\<username\>/main.db
  * Calls
  * Chats
  * Conversations
  * Messages
  * VoiceMails

### Snapchat
* package is com.snapchat.android
* /cache/stories/received/thumbnail
* /sdcard/Android/data/com.snapchat.android/cache/my_media
* /databases/tcspahn.db
  * Chat
  * Conversation
  * ReceivedSnaps
  * SentSnaps

### WhatsApp
* com.whatsapp
* /databases/msgstore.db
  * chat_list
  * messages
* /sdcard/WhatsApp/Media

* On Android, messages are stored encrypted. See [Decrypting WhatsApp Backups](https://blog.elcomsoft.com/2018/12/a-new-method-for-decrypting-whatsapp-backups/) for more information.
* On iOS, if you have access to a phone backup, the messages are not encrypted! This is likely because the developers assumed the iOS platform was more secure, which is a mistake. Also, see this [interesting iOS article](https://www.zdziarski.com/blog/?p=6143) about WhatsApp artifacts.


### Kik
* kik.android
* /database/kikDatabase.db
  * messagesTable
* /sdcard/Kik – media sent and received.

### WeChat
* com.tencent.mm
* The /sdcard/tencent/MicroMsg/*/ folder contain:
  * /video
  * /voice
  * /voice2
* Whereas the /MicroMsg/*/EnMicroMsg.db contains the messages.

The messages are encrypted. See the [Decrypt WeChat Database article](https://articles.forensicfocus.com/2014/10/01/decrypt-wechat-enmicromsgdb-database/) for more information.


# Recovering Deleted Data

Most of the above data is located inside sqlite files. For sqlite, there are unallocated blocks and free blocks. Like a typical file system, when you remove an entity it's simply marked as free but not overwritten immediately. That means you can use a Hex viewer that also shows ASCII to search for keywords that may still be present. File carving is the process of finding and extracting data from storage where you do not have all the information about the file.

There's some valuable information regarding sqlite file carving [here](https://forensicsfromthesausagefactory.blogspot.com/2011/04/carving-sqlite-databases-from.html).

[Scalpel](https://github.com/sleuthkit/scalpel) is an open-sourced data carving tool.

A good tool for viewing an sqlite database is [sqliteviewer](http://www.oxygen-forensic.com/en/features/sqliteviewer/).

Besides message databases, another important thing to look for are pictures. For example, in the JPEG format the first two bytes and last two bytes are always **FF D8** and **FF D9**.

[DiskDigger](https://diskdigger.org/android) is an automated undelete tool available for Android. It scans the device for photos, documents, music and videos.
  
# Where to Read More

So that's a brief overview of the forensics process. For more information check out the following:

* [JTAG and Chip-off methods](https://hitcon.org/2016/CMT/slide/day1-r1-d-1.pdf)
* [Autopsy - The open source digital forensics platform](https://www.autopsy.com/)
* [Santoku Linux - Mobile and malware forensics](https://santoku-linux.com/)
           


