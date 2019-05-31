---
layout: post
category : lessons
tagline: "Securely wiping the contents of memory"
description: Securely wiping the contents of memory on the Apple platform. By Kolin Stürt
tags : [Core Foundation, secure erase, wipe, shred, ios, MAC OS X, tutorial]
---
{% include JB/setup %}

## Securely Wiping the Contents of Memory

In other chapters we will cover securing data in storage and in transit. Sensitive information in memory such as a password or credit card information can be securely wiped when no longer being used. This way an attacker can not access the information by examining the contents of memory while the application is running.

*Disclaimer: While protecting data in storage and in transit is both beneficial and practical, protecting data in memory turns out to be a challenge. One can attempt at obfuscating or encrypting memory contents, but at some point that memory contents must be decrypted in memory in order to be worked upon. All we can really do is try to securely erase the unencrypted information once we are finished with it. Generally speaking, if an attacker either has physical access or remote access to a device in which they have debug privileges or can run code, that sensitive information can easily be compromised without thorough work being done to hide this information prior to an attack.*

With that being said, to wipe the contents of memory we wipe blocks of memory in the same way we would shred a file; by writing over the contents with random data, zeros, or both. We could do this using the memset() function, however, a memset call can be optimized out by the compiler if it notices we are not ever using the contents of the memory we have just set. A better way might be to implement the memset function ourselves, which iterates over the bytes in memory, rewriting it to some other value. By implementing the function ourselves we can control our code from being optimized out. In order to do this we use the *volatile* keyword. This tells the compiler that a variable may have changed without it knowing. The compiler will not cache the value, nor will it optimize out access to the value.

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

Now we can use this function to wipe memory. For any CFDataRef or NSData objects, we can access the bytes pointer by using the CFDataGetBytePtr() function. For strings in memory, we will need to wipe the underlying C pointer which we can get from the CFStringGetCStringPtr() function. When it comes to working with strings containing sensitive information, using the C string primitive data-type, char, is much safer than any higher level CFString or NSString object. These string objects can potentially copy the buffer internally and we would have no way of wiping the copied contents. However, if you need to wipe a string object, the code would look something like this:

	NSString *string = @"hi";
	unsigned char *stringChars = (unsigned char *)CFStringGetCStringPtr((CFStringRef)string, 	CFStringGetSystemEncoding());
	SafeMemset(stringChars, 0, [string length]);
	
	
We could even write helper functions as such

	void SecureWipeString(NSString *string)
	{
	    unsigned char *keyStringChars = (unsigned char *)CFStringGetCStringPtr((CFStringRef)string, CFStringGetSystemEncoding());
	    SafeMemset(keyStringChars, 0, [string length]);
	}
	
	void SecureWipeData(NSData *data)
	{
	    SafeMemset((void *)[data bytes], 0, [data length]);
	}
	
One final note: Be careful clearing the underlying C pointer of a CFStringRef or NSString. On a iDevice for example, if the string contains the word "password", the underlying C pointer just reuses or points to the same address as used by the system; the OS uses the string "password" already so you will crash the application by trying to wipe this area of memory that is outside the sandbox.

Again, to be safe you may want to use a char array, not the char pointer, to store your sensitive strings and wipe them after without ever putting the information into an NSString object. 

