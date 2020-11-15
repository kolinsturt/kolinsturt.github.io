---
layout: post
category : lessons
tagline: "XCode project settings for security"
title: XCode Security Project Settings
description: XCode project settings for app hardening
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
redirect_from:
  - /lessons/2013/03/102/security_related_project_settings
---
{% include JB/setup %}

## XCode Security Project Settings

In recent years, Xcode has built in platform and compiler support for recent security features. Most are enabled by default on new projects. If you're working on an old project, it's a good idea review if they're enabled.

### Address Space Layout Randomization

The OS can place position independent code in primary memory and execute it without problems, regardless of what the absolute address is. It uses different locations for the stack, heap, and executable code for the application each time it launches. That way it's difficult to predict where the code resides in memory at runtime. This complicates performing code exploits. In the example of a buffer overflow, it's not trivial to know where buffers are in memory. To explicitly turn this feature on, set the `-pie` flag in the **Other Linker Flags** area of the Xcode **Build Settings**.

### Stack Smashing Protection

SSP checks for buffer overflows, most importantly for stack smashing attacks. When entering functions that contain vulnerable objects such as buffers, it adds a guard variable containing a random number . It checks the variable when the function exits. Most stack smashing attacks overwrite the return address including the guard variable. If the variable doesn't match it detects this as a corruption.

You can enable SSP in Xcode by adding the `-fstack-protector-all` flag in the **Compiler Flags** section of Xcode. Go to the project settings and choose the **Build Phases** tab. Expand the section for the **Compile Sources** and double click on a file under the **Compiler Flags** section to bring up a textbox to enter the flag.

You can hold shift to highlight multiple entries and hit enter to open a textbox to change the compiler flags for all selected entries. Note that this will overwrite previous flags with the new text.

### Buffer overflows

If you are using GCC, you can add the `_FORTIFY_SOURCE` Preprocessor Macro which checks for particular buffer overflows. Check out the in-depth [description](http://gcc.gnu.org/ml/gcc-patches/2004-09/msg02055.html) for what overflows it prevents.

### Non-Executable Stack and Heap

Newer processors can mark parts of memory as non-executable, set by the NX bit. Both iOS and OS X use this feature by default.

### Optimizations and Obfuscation

Programs written in Objective-C are easy to reverse-engineer. That's due to the dynamic nature of the runtime and the verbosity of method names and strings in the binary. Some solutions are to rename sensitive methods to something innocent, use C so function names aren't in the binary or encrypt string literals when possible. Another option is to use optimizations.

Optimizations are not meant for security, but it can help deter a casual attacker to move onto an easier target. Optimizations can add obfuscation. Security by obscurity is not encryption.

While obfuscation only adds a simple layer to the process of reverse engineering, there are some XCode settings that help protect malicious users disassembling your code.

Apple’s optimizer modifies the code to speed up routines and decrease the size of the compiled output. In your project settings, select the **Build Settings** tab and search for **optimization**. You should see an **Optimization Level** field. Setting the optimization level to `Fastest -O3` may help obfuscate the code. 

You'll want to check the assembly to make sure it worked. In some cases, optimizations can expose the code in a clearer way and make it easier to disassemble. For more information about this setting, see the [Compiler-Level Optimizations](https://developer.apple.com/library/mac/documentation/General/Conceptual/MOSXAppProgrammingGuide/Performance/Performance.html#//apple_ref/doc/uid/TP40010543-CH9-102307) section of the Apple reference guide.

Make sure to test your app using the optimization level set for production. Xcode sets the optimization level differently for the release scheme of your app than for the debug scheme. Change the **Optimization Level** for debug in that **Code Generation** section of the build settings, or go to **Product** > **Scheme** > **Edit Scheme** and in the info tab set the **Build Configuration** to **Release**. Make sure the app works correctly with this setting before you release it.

For more information about obfuscation and app hardening, see the [Secure Coding in Swift](http://code.tutsplus.com/tutorials/secure-coding-in-swift-4--cms-29835) and [Getting Started with ProGuard](https://www.raywenderlich.com/7449-getting-started-with-proguard) tutorials.
