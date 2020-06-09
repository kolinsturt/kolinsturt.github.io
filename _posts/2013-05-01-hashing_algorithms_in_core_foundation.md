---
layout: post
category : lessons
tagline: "CommonCrypto Hash and Message Digest algorithms"
title: Hasing in Common Crypto
description: Hashing algorithms using Common Crypto
author: Kolin Stürt
tags : [CommonCrypto, hash, SHA, message digest, ios, Core Foundation, tutorial]
---
{% include JB/setup %}

## Hashing Algorithms in Core Foundation

In this article you'll create a SHA message digest in the Core Foundation environment. SHA, Secure Hash Algorithm refers to a group of hash functions that often correspond by a version number. SHA-1, similar to the MD5 algorithm uses a 160-bit hash function. SHA-2 included 256 and 512 block sizes. while SHA-3 uses the same block sizes as SHA-2, it was completely redesigned from previous SHA versions based off of predicated attacks. SHA-0, the original version had a serious flaw that SHA-1 fixed. Security researchers also discovered weaknesses in SHA-1 so SHA-2 replaced that version. Another popular hash algorithm was MD5, which was also found to have significant flaws. Therefor, you should use either SHA-2 and SHA-3.

First you'll need to include the *CommonDigest* framework:

	#import <CommonCrypto/CommonDigest.h>

The hash function will take a block of data, in this case a string, and will output a fixed size of data. Developers call the result either a hash or a message digest. In this context these two terms mean the same thing. A hash function is a one-way process. One can not derive the original message given a particular hash. Therefor, hashes are quite good for authentication such as in digital signatures. Modifying a message would change the resulting hash so hashes also have wide use in verification of integrity such as in using checksums. Most login systems will hash the password a user types and match it against the stored hash. This way if attackers compromise the hash table, they can not derive the passwords.

It's best practice to use a block size of either 256 or 512. For this example, you're going to use 512. Define an SHA helper function. 

	CFStringRef Sha512FromString(CFStringRef theString);
	
	
You'll take a string and turn it into raw C characters using the `CFStringGetCStringPtr()` function: 

    uint8_t shaHash[CC_SHA512_DIGEST_LENGTH];
    const char *stringChar = CFStringGetCStringPtr(theString, kCFStringEncodingUTF8);
    CC_LONG stringLength = CFStringGetLength(theString);

The `shaHash` array will store the resulting hash. 

You can use the CommonCrypto functions as one-shot stateless operations. For example, you can call the `CC_SHA512()` function. What this function actually does is initialize, update and finalize the hash. For versatility you can use separate functions instead of a one-shot function. For example, if you have a data stream and are adding chunks of data as they arrive. When your done appending all the bits, you'll need to finalize the data. To do it this way, set up and initialize a context:

	CC_SHA512_CTX sha512Context;
    bzero(&sha512Context, sizeof(sha512Context));
	CC_SHA512_Init(&sha512Context);

Call the update function to add data to a buffer. You can call it more than once:
	
	CC_SHA512_Update(&sha512Context, stringChar, stringLength);
	
After you've added all the data, call the final function to finish the operation and output the result:

	CC_SHA512_Final(shaHash, &sha512Context);


Now you have a message digest. While you can pass it around in a data object, you'll turn it into a string. One trick to turn this into a string might be to package the bytes into a `CFDataRef` object and then run `CFCopyDescription()` on that data. This function reveals hexadecimal digits but it also outputs a lot more information about the object type. You would need to parse the string to get at the information you want. (The same with Foundation's `description`: You'd need to get rid of the spaces and *<* and *>* that are in the output). A better way is to loop through the data and output the correct hex digits:

	// Convert the array of bytes into a string showing its hex representation.
	CFMutableStringRef hashMutableString = CFStringCreateMutable(kCFAllocatorDefault, 	CC_SHA512_DIGEST_LENGTH);
	 
	for (CFIndex i = 0; i < CC_SHA512_DIGEST_LENGTH; i++)
	 {
	     // Add a dash every four bytes, for readability.
	     if (i != 0 && i%4 == 0)
	     {
	         CFStringAppend(hashMutableString, CFSTR("-"));
	     }
	     
	     CFStringAppendFormat(hashMutableString, 0, CFSTR("%02x"), shaHash[i]);
	 }
	 
Now you have a message digest string. You can replace any `CC_` hash function with whichever hash function you need:
	 	
	 	CC_MD2()
	 	CC_MD4()
		CC_MD5()
		CC_SHA1()
		CC_SHA224()
		CC_SHA256()
		CC_SHA384()
		CC_SHA512()
		
Here is the example source code for this tutorial in an XCode project [here](https://github.com/CollinBStuart/MessageDigest).
