---
layout: post
category : lessons
tagline: "Securely wiping strings and data"
title: Secure Wipe Memory
description: Secure Wiping the Contents of Memory in Objective-C
author: Kolin Stürt
tags : [Core Foundation, secure erase, wipe, shred, ios, Objective-C, sensitive strings]
---
{% include JB/setup %}

## Secure Wiping the Contents of Memory

You can attempt to wipe sensitive information in memory such as a password or credit card information. This way an attacker can't access the information by examining the contents of memory while the application is running.

Disclaimer: While protecting data in storage and in transit is both beneficial and practical, protecting data in memory is a challenge. You can attempt at obfuscating or encrypting memory contents but at some point you must decrypt that memory contents to work with it. If an attacker gains access to memory contents, that sensitive information is difficult to secure.

You wipe the contents of memory in the same way you'd shred a file; by writing over the contents with random data, zeros, or both. 

You could try this using the `memset()` function, however, the compiler can optimize out a `memset` call if it notices you're not ever using the contents of the memory just set. 

A better way is to implement a memset function yourself. Your function will iterates over the bytes in memory, rewriting it to another value. 

To prevent the compiler optimizing away your code, use the `volatile` keyword. This tells the compiler that a variable may have changed without it knowing. The compiler will not cache the value, nor will it optimize out access to the value.

Here is the safer memset function:

	void *SafeMemset(void *v, int c, size_t n)
	{
	    //volatile in this case announces that the value might changed behind our back so prevents the compiler from caching it in a CPU register.
	    volatile char *p = (volatile char *)v;
	    while (n--)
	    {
	        *p++ = (char)c;
	    }
	    return v;
	}

Now you can use this function to wipe memory. 

For `NSData` or `CFDataRef` objects, you can access the bytes pointer using the `CFDataGetBytePtr()` function. For strings, you'll need to wipe the underlying C pointer which you can get from the `CFStringGetCStringPtr()` function. 

When it comes to working with strings containing sensitive information, using the C string primitive data-type `char` is much safer than any higher level `NSString` or `CFString` object. That's because string objects often copy the buffer internally and you'd have no way of wiping the copied contents. However, if you need to wipe a string object, here's the code for that:

	NSString *string = @"hi";
	unsigned char *stringChars = (unsigned char *)CFStringGetCStringPtr((CFStringRef)string, 	CFStringGetSystemEncoding());
	SafeMemset(stringChars, 0, [string length]);
	
	
You can put that into a helper function:

	void SecureWipeString(NSString *string)
	{
	    unsigned char *keyStringChars = (unsigned char *)CFStringGetCStringPtr((CFStringRef)string, CFStringGetSystemEncoding());
	    SafeMemset(keyStringChars, 0, [string length]);
	}
	
	void SecureWipeData(NSData *data)
	{
	    SafeMemset((void *)[data bytes], 0, [data length]);
	}
	
Be careful clearing the underlying C pointer of `NSString` or `CFStringRef` when they point to string literals.

On some iOS versions I found that if the string contained system words such as “password”, the underlying C pointer reuses and points to the same address as used by the system. The OS uses the string “password” already so it crashed the application by trying to wipe an area of memory that is outside the sandbox.

To be safe you may want to use a char array, not the char pointer, to store your sensitive strings and wipe them after without ever putting the information into an NSString object.

For more details about secure programming, check out the Objective-C [series](https://kolinsturt.github.io/lessons/2013/03/04/validation), the [Secure Coding in Swift](http://code.tutsplus.com/tutorials/secure-coding-in-swift-4--cms-29835) and [App Hardening Tutorial for Kotlin](https://www.raywenderlich.com/6294778-app-hardening-tutorial-for-android-with-kotlin).
