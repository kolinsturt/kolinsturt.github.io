---
layout: post
category : lessons
tagline: "XCode project settings"
title: XCode project settings for security
description: App hardening - security related XCode project settings
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
---
{% include JB/setup %}

## XCode Security Project Settings

### Obfuscation

Programs written in Objective-C are quite easy to reverse-engineer. The code is quite readable at runtime and some may say this is one of the cons of Objective-C. While obfuscation only adds another simple layer to the process of reverse engineering, there are some XCode settings that while the main goal is to secure against vulnerability, help protect us from malicious users trying to disassemble the code.

The XCode optimizer can either expose part of the code, or in most cases it can help obfuscate your code during disassembly. Apple's optimizer tries to change up the code in order to speed up routines or to decrease the size of the compiled code. In your project settings, select the "Build Settings" tab, and search for "optimization". You should see an Optimization Level field. Setting the optimization level to Fastest -O3 may help obfuscate the code. In some cases it can make it easier to disassemble. Trial and error is needed (for example by using otool). For more information about this setting, see the [Compiler-Level Optimizations](https://developer.apple.com/library/mac/documentation/General/Conceptual/MOSXAppProgrammingGuide/Performance/Performance.html#//apple_ref/doc/uid/TP40010543-CH9-102307) 
section of the Apple reference guide.

**Note: Make sure to test your app using the optimization level you would like to use for production. XCode sets the optimization level differently for the release scheme of your app than for the debug scheme. Change the "Optimization Level" for debug in the Code Generation section of the build settings, or go to Product -> Scheme -> Edit Scheme and in the info tab set the Build Configuration to release. Make sure you app works correctly in this setting before it reaches the public**

### Buffer overflows 

If you are using GCC, you can add the _FORTIFY_SOURCE Preprocessor Macro which checks for particular buffer overflows. You can find an in-depth description of what overflows it prevents [here](http://gcc.gnu.org/ml/gcc-patches/2004-09/msg02055.html)

### Stack Smashing Protection

SSP checks for buffer overflows, most importantly for stack smashing attacks. It adds a guard variable containing a random number when entering functions that contain vulnerable objects such as buffers. Then when the function exits, the variable is checked. Most stack smashing attacks will overwrite the return address, including the guard variable so it is possible to detect corruption.

You can enable SSP in XCode by adding the -fstack-protector-all flag in the Compiler Flags section of XCode. Go to the project settings and choose the "Build Phases" tab. Expand the section for the "Compile Sources" and double click on any file under the "Compiler Flags" section to bring up a textbox to enter the flag.

TIP: You can hold shift to highlight multiple entries and hit enter to open a textbox to change the compiler flags for all selected entries. However, any previous flags will be overwritten with the new text.

### Address Space Layout Randomization
Position independent code can be placed in primary memory and will execute without problems regardless of what their absolute address is. Different locations are used for the stack, heap, and executable code for the application each time it launches so that it is difficult to figure out where the code in memory will actually be at runtime. This complicates performing buffer overflows as it is not easily possible to know where buffers are in memory and where code is specifically located. To explicitly turn this feature on, set the -pie flag in the "Other Linker Flags" area of XCode's "Build Settings".


### Non-Executable Stack and Heap
Newer processors can mark parts of memory as non-executable. This is set by what is called an NX bit. Both iOS and OS X use this feature by default.

### Static Code Analysis
Last but not least, be sure to always run the static analyzer on your code, as it checks for security issues such as format string attacks and functions that are susceptible to buffer overflows. You can run the static analyzer from the Product menu in XCode. Different security analyzation options can be set by selecting the project settings in XCode, navigation to the "Build Settings" tab. In the search textbox, type Security. A group will appear called "Static Analyzer - Issues - Security". From here you can turn on checking of specific issues such as "Use of 'rand' functions"  (which has [vulnerabilities](http://www.tech-faq.com/random-number-vulnerability.html)) instead of the newer arc4rand() or SecRandomCopyBytes() functions.
