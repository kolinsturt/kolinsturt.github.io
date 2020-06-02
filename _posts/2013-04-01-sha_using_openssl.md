---
layout: post
category : lessons
tagline: "OpenSSL SHA Hash"
title: SHA in OpenSSL
description: SHA Hash in OpenSSL
author: Kolin Stürt
tags : [OpenSSL, C++, hash, SHA, message digest, ios, Core Foundation, tutorial]
---
{% include JB/setup %}

## SHA using OpenSSL

In this post you're going to hash data using Secure Hash Algorithm - SHA. SHA is a one-way message digest mechanism that actually refers to a group of hash functions that often correspond by a version number. SHA-1 produces 160 bit, or 20 byte output and is similar to MD4 and MD5. SHA-2 included 256 and 512 block sizes, while SHA-3 uses the same block sizes as SHA-2, but was completely redesigned from previous SHA versions based off of predicated attacks. Generally you want to use 256 or 512 block sizes as it's more secure. In [this tutorial](https://collinbstuart.github.io/lessons/2013/05/01/hashing_algorithms_in_core_foundation/), I covered using SHA512 using the Common Crypto library. 

SHA-1 has major weaknesses. It is not approved for production cryptographic implementations. You should not for applications relying on security. [1](https://www.schneier.com/blog/archives/2005/02/cryptanalysis_o.html) [2](http://2012.sharcs.org/slides/stevens.pdf). However, SHA-1 is most widely used and often needed for compatibility such as communication with a server that you do not control, so for this tutorial we will use SHA-1.

You'll use OpenSSL for the encryption library. While OpenSSL is C code, it is quite common to use it for C++ development. The other common library is [Crypto++](http://www.cryptopp.com/). You can use the one-shot SHA1() function, or if the entirety of the message is not stored in memory, you can use multiple functions. We will demonstrate the use of multiple functions for this example.

### Initializing a Context

To initialize a context structure, call SHA1_Init(). You then add chunks using the SHA1_Update() function. When there's no more data to add, call SHA1_Final() to get a pointer to the hashed data. This function will also erase the context structure.

There are higher level functions for message digests - [EVP_Digest](https://www.openssl.org/docs/crypto/EVP_DigestInit.html), however for this example you're going to stick to direct access to the hash functions. Create a header file and call it MessageDigest.h

	#include <stdio.h>
	#include <string>
	#include <vector>
	
	using namespace std;
	
	class MessageDigest
	{
	public:
	    static string messageDigestStringFromData(vector<uint8_t> data);
	};

### Performing the Hash Function

Create a static function to hash data, then turn that data into a string so that you can easily store or transmit it. The string will reveal the hexadecimal digits for the hashed data. Lets add this to the implementation file, MessageDigest.cpp. Also don't forget to link against the libcrypto and libssl libraries.

	#import <iostream>
	#include <openssl/sha.h>
	
	string MessageDigest::messageDigestStringFromData(vector<uint8_t> data)
	{
	    string resultString;
	    
	    if ( ! data.empty())
	    {
	        unsigned char sha1Hash[SHA_DIGEST_LENGTH];
	        SHA_CTX context;
	        bzero(&context, sizeof(context));
	        bool success = SHA1_Init(&context);
	        if (success)
	        {
	            success = SHA1_Update(&context, data.data(), data.size());
	        }
	        if (success)
	        {
	            success = SHA1_Final(sha1Hash, &context);
	        }
	        
	        
	        if (success)
	        {
	            resultString.reserve(SHA_DIGEST_LENGTH);
	            for (std::size_t i = 0; i != SHA_DIGEST_LENGTH; ++i)
	            {
	                resultString += "0123456789abcdef"[sha1Hash[i] / 16];
	                resultString += "0123456789abcdef"[sha1Hash[i] % 16];
	            }
	        }
	    }
	    
	    return resultString;
	}

### Creating a Wrapper

That’s all you need to do to create a message digest. Since the focus is on iOS and OS X platforms, write a function to make it interoperable with Core Foundation, and thus the Foundation Cocoa environment. Here is an updated header file, adding the Core Foundation framework and one new static function that returns a CFStringRef.

	#include <stdio.h>
	#include <string>
	#include <vector>
	#include <CoreFoundation/CoreFoundation.h>
	
	using namespace std;
	
	class MessageDigest
	{
	public:
	    static string messageDigestStringFromData(vector<uint8_t> data);
	    static CFStringRef createMessageDigestStringFromData(CFDataRef data); //need to CFRelease data
	};



You will need to add the appropriate headers in your implementation file:

	#import <CommonCrypto/CommonDigest.h>
	#import <CommonCrypto/CommonCryptor.h>
	#import <Security/Security.h>
	
Here's a Core Foundation method to add to MessageDigest.cpp:

	CFStringRef MessageDigest::createMessageDigestStringFromData(CFDataRef chunkData)
	{
	    string resultString;
	    
	    if (chunkData)
	    {
	        CFIndex chunkDataLength = CFDataGetLength(chunkData);
	        vector<uint8_t> vector;
	        vector.resize(chunkDataLength);
	        CFDataGetBytes(chunkData, CFRangeMake(0, chunkDataLength), &vector[0]);
	        resultString = messageDigestStringFromData(vector);
	    }
	    
	    /* This would be the full “Common Crypto Way”, but is much less portable
	    uint8_t sha1Hash[CC_SHA1_DIGEST_LENGTH];
	    CC_SHA1_CTX sha1Context;
	    bzero(&sha1Context, sizeof(sha1Context));
	    CC_SHA1_Init(&sha1Context);
	    CC_SHA1_Update(&sha1Context, CFDataGetBytePtr(chunkData), (CC_LONG)chunkDataLength);
	    CC_SHA1_Final(sha1Hash, &sha1Context);
	    CFMutableStringRef messageDigestString = CFStringCreateMutable(kCFAllocatorDefault, 0);
	    CFDataRef hashData = CFDataCreate(kCFAllocatorDefault, sha1Hash, CC_SHA1_DIGEST_LENGTH);
	    CFStringRef hashRepresentationString = CFCopyDescription(hashData);
	    CFStringAppend(messageDigestString, hashRepresentationString);
	    CFRelease(hashRepresentationString);
	    CFRelease(hashData);
	    
	    string hashString = CFStringGetCStringPtr(messageDigestString, kCFStringEncodingUTF8);
	    CFRelease(messageDigestString);
	    
	    string findString = "bytes = 0x";
	    unsigned long location = hashString.find(findString);
	    if (location != string::npos)
	    {
	        hashString = hashString.substr(location + findString.length(), 40);
	    }
	     */
	    
	    return CFStringCreateWithCString(kCFAllocatorDefault, resultString.c_str(), kCFStringEncodingUTF8);
	}

You used the word “create” at the beginning of the function name; denoting that the returned object has a retain count of +1 and will need a CFRelease();

Here is the full [MessageDigest](https://github.com/CollinBStuart/SHAOpenSSL) class for you to download.


