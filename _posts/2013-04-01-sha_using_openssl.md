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

SHA-1 has major weaknesses. It is not approved for production cryptographic implementations. You should not for applications relying on security. [1](https://www.schneier.com/blog/archives/2005/02/cryptanalysis_o.html) [2](http://2012.sharcs.org/slides/stevens.pdf).

You'll use the SHA2 family of operations in the OpenSSL  encryption library. While OpenSSL is C code, it is quite common to use it for C++ development. The other common library is [Crypto++](http://www.cryptopp.com/). You can use one-shot functinos such as `SHA512`. This is good if you already have all the data to hash. If the entirety of the message is not stored in memory such as when using streams, you can use multiple hash calls. You'll learn this way because the solution is more extensible. It works for both streams and one-shot logic.

### Initializing a Context

To initialize a context structure, call `EVP_DigestInit()`. You then add chunks using the `EVP_DigestUpdate()` function. When there's no more data to add, call `EVP_DigestFinal()` to get a pointer to the hashed data. This function will also erase the context structure.

These are higher level functions for message digests - [EVP_Digest](https://www.openssl.org/docs/crypto/EVP_DigestInit.html). The EVP methods are "envelope", high level interfaces to cryptographic operations. Always start with a high level solution. Low level solutions may introduce more security holes if you're not careful with the implementation. Create a header file and call it *MessageDigest.h*. Don't forget to link against the libcrypto and libssl libraries:

        #include <iostream>
	#include <openssl/sha.h>
	#include <stdio.h>
	#include <string>
	#include <vector>
	
	using namespace std;
	
	class MessageDigest
	{
	public:
	    static string sha2ForString(std::string &theString, int length);
	    static string messageDigestStringFromData(vector<uint8_t> data);
	};

### Creating a Digest

First, you'll hash a `string` in one go by using OpenSSL's EVP API. Create a function to hash the `string` given a specific hash length:

    std::string MessageDigest::sha2ForString(std::string &theString, int length)
    {
        string shaString;
        if ( ! theString.empty())
        {
            EVP_MD_CTX *context = EVP_MD_CTX_create();
            EVP_MD_CTX_init(context);
            if (context)
            {
                EVP_MD *type = NULL;
                switch (length)
                {
                    case SHA512_DIGEST_LENGTH:
                    {
                        type = (EVP_MD *)EVP_sha512();
                        break;
                    }
                    case SHA384_DIGEST_LENGTH:
                    {
                        type = (EVP_MD *)EVP_sha384();
                        break;
                    }
                    case SHA256_DIGEST_LENGTH:
                    {
                        type = (EVP_MD *)EVP_sha256();
                        break;
                    }
                    case SHA224_DIGEST_LENGTH:
                    {
                        type = (EVP_MD *)EVP_sha224();
                        break;
                    }
                    default:
                    {
                        type = (EVP_MD *)EVP_sha512();
                        break;
                    }
                }
                
                if (EVP_DigestInit_ex(context, type, NULL))
                {
                    if (EVP_DigestUpdate(context, theString.c_str(), theString.length()))
                    {
                        unsigned char hash[EVP_MAX_MD_SIZE];
                        unsigned int lengthOfHash = 0;

                        if (EVP_DigestFinal_ex(context, hash, &lengthOfHash))
                        {
                            std::stringstream ss;
                            for(unsigned int i = 0; i < lengthOfHash; ++i)
                            {
                                ss << std::hex << std::setw(2) << std::setfill('0') << (int)hash[i];
                            }

                            shaString = ss.str();
                        }
                    }
                }
                EVP_MD_CTX_cleanup(context);
                EVP_MD_CTX_destroy(context);
            }
        }
        return shaString;
    }

In this code, you initialized a `EVP_MD_CTX` and `EVP_MD type` given one of four lengths: 224, 256, 384 and 512. After you create the hash, you cleanup the contexts by calling `EVP_MD_CTX_cleanup` and `EVP_MD_CTX_destroy`.

### Performing Low Level Operations

You can use one-shot functinos such as `SHA512` or it's init, update and final counterparts. This time you'll create a static function to hash *binary* data, then turn that data into a string so that you can easily store or transmit it. The string will reveal the hexadecimal digits for the hashed data. Add this to the implementation file *MessageDigest.cpp*:
	
	string MessageDigest::messageDigestStringFromData(vector<uint8_t> data)
	{
	    string resultString;
	    
	    if ( ! data.empty())
	    {
	        array<unsigned char, SHA512_DIGEST_LENGTH> shaHash;
                SHA512_CTX context;
                bzero(&context, sizeof(context));
                bool success = SHA512_Init(&context);
                if (success)
                {
                    success = SHA512_Update(&context, CFDataGetBytePtr(chunkData), chunkDataLength);
                }
                
		if (success)
                {
                    success = SHA512_Final(shaHash.data(), &context);
                }
            
                if (success)
                {
                    if ( ! shaHash.empty())
                    {
                        //Convert to HEX/Base16
                        resultString.reserve(SHA512_DIGEST_LENGTH);
                        for (std::size_t i = 0; i != shaHash.size(); ++i)
                        {
                            resultString += "0123456789abcdef"[shaHash[i] / 16];
                            resultString += "0123456789abcdef"[shaHash[i] % 16];
                        } //end for
                    } //end if ( ! sha1Hash.empty())
                } //end if (success)
	    }
	    
	    return resultString;
	}

Here you used the SHA512 functions to hash binary data and converted it to a HEX (Base16) string.

### Porting The Code

That’s all you need to do to create a message digest. But if you'd like to port this to iOS and OS X platforms for example, write a function to make it interoperable with Core Foundation, and thus the Foundation Cocoa environment. Here is an updated header file, adding the Core Foundation framework and one new static function that returns a CFStringRef.

	#include <stdio.h>
	#include <string>
	#include <vector>
	#include <CoreFoundation/CoreFoundation.h>
	
	using namespace std;
	
	class MessageDigest
	{
	public:
	    static string sha2ForString(std::string &theString, int length);
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
	    
	    return CFStringCreateWithCString(kCFAllocatorDefault, resultString.c_str(), kCFStringEncodingUTF8);
	}

You used the word “create” at the beginning of the function name; denoting that the returned object has a retain count of +1 and will need a CFRelease();

Here is the full [MessageDigest](https://github.com/CollinBStuart/SHAOpenSSL) class for you to download.


