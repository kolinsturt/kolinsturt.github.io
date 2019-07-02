---
layout: post
category : lessons
tagline: "Reverse Engineering Android Apps"
description: Tools to aid in reverse engineering Android apps - Kolin Stürt.
tags : [Android, aapt, DEX, Drozer, Dex2Jar, jd_gui, smali, baksmali]
---
{% include JB/setup %}

# Beginning Android Forensics

The main challenge in forensics recovery is dealing with a secured device. For Android, almost all solutions involve rooting the device to gain access. There are many tools to root a device, such as [OneClickRoot](https://www.oneclickroot.com/), [KingoRoot](https://www.kingoapp.com/) and [SuperUserDownload](https://superuserdownload.com/).

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



# Reverse Engineering

After a logical or physical aquisition of a device is made, you'll want to look at user data stored inside each app. Sometimes apps use obfuscated or eccentric ways of storing the user's data so the focus shifts to understanding the particular app and how it works internally.

There's been lots of discussion about reverse engineering for apps written in Java. Recently Kotlin has become popular as an alternative to Android app development. The good news is that Kotlin is a JVM language. When you build your app, Android Studio produces an APK file. This is really just a zip file, and has a structure similar to **jar** Java archives. Inside the archive are resources along with a DEX file. DEX stands for Dalvik Executable. 

Apps run on a Java Virtual Machine (JVM). Traditionally on Android, the JVM used was Dalvik. However in recent years, ART, the Android Runtime has replaced Dalvik. It converts the DEX into native code for performance by running a tool called Dex2Oat that creates a native ELF binary.

While Kotlin has it’s own syntax, the **kotlinc** compiler just transforms the code into a DEX file that contains Java bytecode. So because Kotlin is compiled to the same bytecode as Java, that means that most of the reverse engineering tools are the same as they are for apps built in Java. 

With that being said, here is a standard process for reverse engineering Android apps:

# adb

First take the release build from Android studio, or alternatively use adb shell to list all available apps

    adb shell pm list packages | grep YourAppName


# aapt

Android Asset Packaging Tools can be used to dump the Android Manifest file

    aapt dump xmltree /appFolder/app-release.apk AndroidManifest.xml

as well as resource and asset files included in the APK

    aapt l -a /appFolder/app-release.apk


# AXMLPrinter2

First, you can unzip an apk easily like this

    unzip app-release.apk

Next, you can use [AXMLPrinter2](https://code.google.com/archive/p/android4me/downloads) to parse Android binary XML formats directly. For example, to look at the Android Manifest file

    java -jar AXMLPrinter2.jar AndroidManifest.xml

# Smali

[smali/baksmali](https://github.com/JesusFreke/smali) is an assembler and disassembler for the DEX format that is used by Dalvik. Baksmali will disassemble the APK file into the Jasmin syntax but one thing about this tool is that it can take the ProGaurded obfuscated names and unravel them so you can see the names of the methods. This means it is a good idea to still name sensitive methods with something more innocent.

    java -jar baksmali-2.1.2.jar app-release.apk

Files are outputed to a /out folder. You can then use Smali to take the outputed files and convert them into a DEX file.

    java -jar smali-2.1.2.jar -o classes.dex /out/

# Dex2Jar

[Dex2Jar](https://github.com/pxb1988/dex2jar). Dex files created from the above method can then be translated back to something that resembles the original source code. You can convert the DEX file to a standard Java CLASS file.

    d2j-dex2jar.sh /app-release.apk -o /AppName.jar

# JD-GUI

Once you have your jar file from the above method, you can open it to get all the class names and most source code by opening the jar folder in [JD-GUI](https://github.com/java-decompiler/jd-gui).

# IDA Pro
Commercial. You can dissassemble and debug Dalvik code since [IDA Pro v6.1](https://hex-rays.com/products/ida/6.1/index.shtml). IDA is good because of its support for scripting and it has a graph-view which can unwind the flow of the app. There's also lots of scripts people write for it to assist in unwinding obfuscated code.

# JEB

Also commercial. [JEB](https://www.pnfsoftware.com/) can understand ARM and ELF formats. It has a powerful UI for both Dalvik and native code.

# Drozer

[Drozer](https://labs.mwrinfosecurity.com/tools/drozer/) allows you to assume the role of an Android app and interact with other apps. One of the modules in drozer, app.package.maifest will parse the manifest file and display it on screen.

    run app.package.maifest com.company.appName

# Others

*[Dextra](http://newandroidbook.com/tools/dextra.html) supports ART and OAT.

*[ApkTool](https://ibotpeaches.github.io/Apktool/) will reverse-engineer the entire Android backage back to a workable form, including all resources and origional source code. 

*[Jadx](https://github.com/skylot/jadx). This will let you browse decompiled DEX code. It also decompiles most of the entire project. 

*[JAD](http://varaneckas.com/jad/). This will convert Java Class files back to source files. 
