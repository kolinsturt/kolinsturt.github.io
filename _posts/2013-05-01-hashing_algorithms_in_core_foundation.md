---
layout: post
category : lessons
tagline: "CommonCrypto Hash and Message Digest algorithms"
description: Hashing algorithms using Common Crypto by Kolin Stürt
tags : [CommonCrypto, hash, SHA, message digest, ios, Core Foundation, tutorial]
---
{% include JB/setup %}

## Hashing Algorithms in Core Foundation

In this tutorial we will look at creating a SHA message digest in the Core Foundation environment. SHA, Secure Hash Algorithm refers to a group of hash functions that often correspond by a version number. SHA-1, similar to the MD5 algorithm uses a 160-bit hash function. SHA-2 included 256 and 512 block sizes. while SHA-3 uses the same block sizes as SHA-2, it was completely redesigned from previous SHA versions based off of predicated attacks. However, those attacks so far have not been realized. SHA-0, the original version was known to have a serious flaw and was replaced by SHA-1. Weaknesses were also discovered in SHA1 so SHA-2 replaced this version. Another popular hash algorithm was MD5, which was also found to have significant flaws. Generally speaking, both SHA-2 and SHA-3 are considered quite robust.

Lets include the CommonDigest framework now

	#import <CommonCrypto/CommonDigest.h>

The hash function will take a block of data, in this case a string, and will output a fixed size of data. The result is called either a hash or a message digest. These two terms mean the same thing and are often used interchangeably. A hash function is a one-way process. One can not derive the original message given a particular hash. Therefor, hashes are quite good for authentication such as in digital signatures. Modifying a message would change the resulting hash so hashes also have wide use in verification of integrity such as in using checksums. Most login systems will hash the password a user types and match it against the stored hash. This way if the hash table is compromised, the password still can not be derived.

**NOTE: MD5, SHA-0 and SHA-1 are known to have major weaknesses. They are not approved for production cryptographic implementations and should not be used generally for applications relying on security.**

The first thing we will do is setup some variables. Our function will have this definition

	CFStringRef Sha512FromString(CFStringRef theString);
	
	
Lets take a string and turn it into its raw C characters. This is fairly easy with the CFStringGetCStringPtr() function. 

    uint8_t shaHash[CC_SHA512_DIGEST_LENGTH];
    const char *stringChar = CFStringGetCStringPtr(theString, kCFStringEncodingUTF8);
    CC_LONG stringLength = CFStringGetLength(theString);

Our shaHash array will store the result hash. For this example, we are going to use the 512 block size function.

In the previous examples, we used the CommonCrypto functions as one-shot stateless operations. We can do this with the CC_SHA512() function. However, what this function actually does, is initialize, update and finalize the hash. These can be used as separate functions instead of a one-shot function if we have set up a stream and are adding multiple bits of data to hash (by setting up a hash context), for example. Then when we are done appending all the bits, we would finalize the data. Lets set up and initialize a context

	CC_SHA512_CTX sha512Context;
    bzero(&sha512Context, sizeof(sha512Context));
	CC_SHA512_Init(&sha512Context);

Next we call the update call to add data to a buffer. This can be called many times.
	
	CC_SHA512_Update(&sha512Context, stringChar, stringLength);
	
After we have added all the data, we call the final function to finish the operation and output the result.

	CC_SHA512_Final(shaHash, &sha512Context);


Now we have a message digest. For this example, we will turn it into a string but it could easily be passed around in a data object. One trick to turn this into a string might be to package the bytes into a CFData object and then run CFCopyDescription() on that data. Like Foundation's description method, this function would reveal hexadecimal digits, but it also outputs a lot more information about the object type so you would need to parse the string to just get at the information you want (The same with Foundation's description, one would need to get rid of the spaces and '<' and '>' that would be added to the text output). A better way would be to loop through the data byte by byte and output the correct hex digits

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
	 
Now we have a message digest string and that is it. Keep in mind you can replace any CC_SHA512 function with whichever hash function you would like. For example
	 	
	 	CC_MD2()
	 	CC_MD4()
		CC_MD5()
		CC_SHA1()
		CC_SHA224()
		CC_SHA256()
		CC_SHA384()
		CC_SHA512()
		
Here is the example source code for this tutorial in an XCode project [here](https://github.com/CollinBStuart/MessageDigest).
