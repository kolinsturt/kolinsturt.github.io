---
layout: post
category : lessons
tagline: "Hacking iOS apps"
title: iOS Forensics Part 2 - Reverse Engineering
author: Kolin Stürt
description: Decrypting, reverse engineering and patching
tags : [iOS, hacking, patching, hooking, reverse engineering, security, XCode]
redirect_from:
  - /lessons/2015/01/01/iOS_hacking
---
{% include JB/setup %}

In the [previous article](https://kolinsturt.github.io/lessons/2016/01/01/beginning_ios_forensics) you looked at beginning iOS forensics in general. This guide looks at how to reverse engineer an app binary.

## Decrypting the Binary

This requires a jailbroken phone with the [OpenSSH Cydia](https://cydia.saurik.com/openssh.html) package installed.

1. Using a jailbroken phone, navigate to **Settings** -> **Wi-Fi** and click the **"i"** icon to get details of the current wifi connection. Note the IP Address (For example, **10.0.0.198**)

2. Launch an SFTP / SSH client such as [Cyberduck](https://cyberduck.io/)

3. Connect to **sftp://root@10.0.0.198:22** (The default root password for all iOS devices is **alpine**. You may want to change that if you'll be leaving OpenSSH active)

4. Since you are not Apple, you will get a warning about an unknown fingerprint, press **OK**


iOS 7 : The user applications are located at the location **/var/mobile/Applications**

iOS 8+ : The application bundle is stored in the location **/var/mobile/Containers/Bundle/Application** (Appname.app) whereas the application data (Documents, Library, tmp folder) is stored in the location **/var/mobile/Containers/Data/Application**. The name of the folder (a unique ID) will also be different for the same application. So while checking an application, it is recommended to look at both the locations.

* note that **/var** may not work, you may need to navigate to **/private/var** to see the "mobile" folder

### iOS 7
* Using [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)

1.  git clone git@github.com:stefanesser/dumpdecrypted.git
```
make
```
2. Upload **dumpdecrypted.dylib** to iphone, then **ssh** to the iphone (`ssh root@10.0.0.198`)
```
iPhone:~ root# DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Containers/Bundle/Application/xx-xxxx-xx/Scan.app/Scan mach-o decryption dumper
```
3. Then **Scan.decrypted** will be saved to current directory. Run this to verify if it's decrypted.
```
iPhone:~ root# class-dump-z Scan.decrypted
```

### iOS 8+
* Using [Clutch](https://github.com/KJCracks/Clutch)
 
1. First disable code signing:

```
killall Xcode
cp /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/SDKSettings.plist ~/
/usr/libexec/PlistBuddy -c "Set :DefaultProperties:CODE_SIGNING_REQUIRED NO" /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/SDKSettings.plist
/usr/libexec/PlistBuddy -c "Set :DefaultProperties:AD_HOC_CODE_SIGNING_ALLOWED YES" /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/SDKSettings.plist
```

2. download and extract Clutch, **cd** into project folder

3. open project and set signing identity to your team

4. close project, in command line 
```
xcodebuild clean build
```
5. Then copy the build file to the phone
```
scp ./build/Clutch root@10.0.0.197:/usr/bin/Clutch
```
6. SSH back into the device
```
ssh root@10.0.0.198
Clutch -b com.appcomp.appName
```
It will output something like **"DONE: /private/var/mobile/Documents/Dumped/x.x.x-iOS8.0-(Clutch-2.0.4)-2.ipa"**

7. [Cyberduck](https://cyberduck.io/) copy the ipa file to your computer

## Reverse Engineering

Here's a quick list of tools that aid in reverse engineering the code:

[nm](https://linux.die.net/man/1/nm)
List symbols from object files.
ex: `nm thelib.a`

[strings](https://linux.die.net/man/1/strings)
Print the strings of printable characters in files. 

[otool](https://www.manpagez.com/man/1/otool/)
Use to examine the Objective-C runtime information stored in the Mach-O file.

[class-dump](http://stevenygard.com/projects/class-dump/) and [class-dump-z](https://code.google.com/archive/p/networkpx/wikis/class_dump_z.wiki)
Uses otool but can see actual Objective-C interfaces and declarations.

[Keychain Dumper](https://github.com/ptoomey3/Keychain-Dumper)
See which keychain items are accessible to an attacker when a device is jailbroken.

[IDA Pro](https://www.hex-rays.com/products/ida/)
Commercial disassembler and debugger.
Standard when it comes to iOS reverse engineering (Also because of https://www.hex-rays.com/products/ida/support/idadoc/1687.shtml)

## Hooking and Patching

In addition to understanding how the code works, you can modify it to intercept function calls - called hooking. You can use this to monitor methods or replace security checks, for example. Here's a list of popular tools to help with the process:

[Cydia Impactor](http://www.cydiaimpactor.com/)
Use this to install IPA files on iOS.

[Frida](https://www.frida.re/)
Allows you to inject JavaScript snippets into iOS apps. It can hook functions and trace private code.

[Cycript](http://www.cycript.org/)
Cycript lets you browse and modify iOS apps at runtime using Objective-C++ and JavaScript.

[Cydia Substrate](http://www.cydiasubstrate.com/)
The very popular code modification platform.

[MSHookFunction](http://www.cydiasubstrate.com/api/c/MSHookFunction/)
Hijack the behavior of native code or a private symbol.

Besides looking at the data stored on the device and in the app, you'll need to analyze the data an app sends and receives over the network. That can give you a lot of data and insight as to how the app works. Check out the [Intercepting Network Traffic](https://kolinsturt.github.io/lessons/2016/06/01/intercepting_network_traffic_ios) article next to learn how to do that.
