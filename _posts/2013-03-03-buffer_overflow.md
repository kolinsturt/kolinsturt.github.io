---
layout: post
category : lessons
tagline: "Buffer Overflows and Format String Attacks"
title: Buffer Overflows and Format String Attacks
description: A discussion of buffer overflows and format string attacks
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
---
{% include JB/setup %}

### Buffer Overflows

Secure coding is the art of writing software so you protect it from security vulnerabilities and malicious attacks. The most common of the vulnerabilities in a C based language appear as buffer overflows, integer overflows, buffer underflows, and format string attacks. For a primer on these subjects, check out Apple’s [Secure Programming Guide](https://developer.apple.com/library/mac/documentation/security/conceptual/SecureCodingGuide/Articles/SecurityGuidelines.html#//apple_ref/doc/uid/TP40009511-SW1). There's a list of common functions susceptible to buffer overflow attacks and which ones to use as replacements:

* strcat -> strlcat
* strcpy -> strlcpy
* strncat -> strlcat
* strncpy -> strlcpy
* sprintf -> snprintf -> asprintf
* vsprintf -> vsnprintf -> vasprintf
* gets -> fgets -> use Core Foundation or Foundation APIs

Apple states in their document:

> Security Note for snprintf and vsnprintf: The functions snprintf, vsnprintf, and variants are 	dangerous if used incorrectly. Although they do behave functionally like strlcat and similar in 	that they limit the bytes written to n-1, the length returned by these functions is the length 	that would have been printed if n were infinite.
	
> For this reason, you must not use this return value to determine where to null-terminate the string or to determine how many bytes to copy from the string at a later time.
	
> Security Note for fgets: Although the fgets function provides the ability to read a limited amount of data, you must be careful when using it. Like the other functions in the “safer” column, fgets always terminates the string. However, unlike the other functions in that column, it takes a maximum number of bytes to read, not a buffer size.
	
> In practical terms, this means that you must always pass a size value that is one fewer than the size of the buffer to leave room for the null termination. If you do not, the fgets function will dutifully terminate the string past the end of your buffer, potentially overwriting whatever byte of data follows it.

If you are writing code in C, you can use the Core Foundation representation of a string, CFStringRef. All of it's string manipulation functions start with CFString and are safe against buffer overflows.

If you are using Foundation, the same is true for the NSString class. NSString and CFString are also toll-free bridged in that they can be casted back and forth between the two.

If you are porting code in C++, std::string is safe from buffer overflows.

If you must use C, here are some examples to help avoid buffer overflows.

Try not to use hard coded buffer sizes such as:

	UInt8 buffer[512];
	if (size <= 511) 
	{
	   ;
	}


It would be better to define the buffer size once:

	#define BUFFER_SIZE 512
	UInt8 buffer[BUFFER_SIZE];
	if (size < BUFFER_SIZE) 
	{
	   ;
	}
	
	
It is even better to do this:

	if (size < sizeof(buffer)) 
	{
	   ;
	}


### Integer Overflows

Make sure the compiler doesn't optimize out your integer overflows. This can happen if the compiler sees checks where you never use the result. For example, here's a check for a potential overflowing multiplication:

    size_t byteSize = (first * second);
    if ( (bytes < first) || (bytes < second) ) //check for overflow
    {
	;
    }

A point from Apple's guide:

>the only correct way to test for integer overflow is to divide the maximum allowable result by the multiplier and comparing the result to the multiplicand or vice-versa. If the result is smaller than the multiplicand, the product of those two values would cause an integer overflow.

Here is an example:

	if (first > 0 && second > 0 && SIZE_MAX / first >= second) 
	{
	    size_t byteSize = (first * second);
	    //...make allocation
	}



### Buffer Underflows

To prevent buffer underflows, zero all buffers before you begin to use them. Do this with `memset()`, `bzero()`, or `calloc()` for example. When allocating or initializing memory, check that the function returned a success code before processing the resulting data. If applicable, use any value returned by a function that tells you the size of the data returned and use this value rather than a hard-coded one. If you have a constant of what the size should have been, you can check this and fail gracefully if you did not receive the expected size instead of going on to process the data. The same should apply to any functions that operate on your data. For example, make sure a write call completes successfully before operating on that data. Do the same for opening and reading data. 

Make sure the size and type of data is what you expect. Object strings such as `NSString`, `CFStringRef` or` std::string` handle size checks themselves, so it is best to leave the string in their object form if possible. Converting objects to primitive C types can cause vulnerabilities. `CFString`’s `CFStringCreateWithCString()` does not take into account the length of the C string buffer. If possible, separate buffer and data operations from string manipulation functions.

`CFDataRef` lets you access the underlying primitive data via `CFDataGetBytes`. You have to manage primitive memory with `malloc` and `free` instead of reference counting. There's the potential to free, overwrite or reuse the buffer accidentally.

Collection classes such as `CFSet`, `CFArray`, and `CFDictionary` create objects so that the user must first pass in the number of objects in the collection. Make sure this number is correct. Some of foundation’s methods let you pass in a list of objects of a variable length with nil as the last element. If you forget to pass in `nil`, the initialize will continue reading into the stack of the process. Xcode gives you a warning if you forget the `nil` at the end. That's why it's a good idea to eliminate all warning so that new ones will stand out. Treat warnings as errors.

`CFArrayGetValues` or `NSArray`'s `getObjects` create a primitive C array. It does not check the length of the buffer you passed in. If the buffer is not big enough it will overflow.

### Format String Attacks

Format string functions are susceptible to format string attacks. `printf`, `CFStringCreateWithFormat` or `NSString`’s `stringWithFormat` are examples. To guard against attacks, make sure that data is not passed as part of the format string. Use your own format string; Don't log variables directly as you don't know if they contain a **%**. For example:

        NSLog(maliciousString);
	
should be:

        NSLog(@"%@", maliciousString);
	
Just as:
	
	printf(buffer)
	
should be:
	
	printf("%s", buffer)


To learn more about secure coding in other environments, check out the [Secure Coding in Swift](http://code.tutsplus.com/tutorials/secure-coding-in-swift-4--cms-29835) and [App Hardening Tutorial for Kotlin](https://www.raywenderlich.com/6294778-app-hardening-tutorial-for-android-with-kotlin).
