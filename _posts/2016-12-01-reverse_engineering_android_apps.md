---
layout: post
category : lessons
tagline: "Reverse Engineering Android Apps"
description: Tools to aid in reverse engineering Android apps - Kolin Stürt.
tags : [Android, aapt, DEX, Drozer, Dex2Jar, jd_gui, smali, baksmali]
---
{% include JB/setup %}


# Reverse Engineering Kotlin
The good news is that Kotlin is a JVM language. When you build your app, Android Studio produces an APK file. This is really just a zip file, and has a structure similar to **jar** Java archives. Inside the archive are resources along with a DEX file. DEX stands for Dalvik Executable. 

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
