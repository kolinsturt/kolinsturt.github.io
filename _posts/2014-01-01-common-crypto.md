---
layout: post
category : lessons
tagline: "Using the CommonCrypto library in CoreFoundation"
title: AES Encryption using Common Crypto
description: Encrypting data using the Common Crypto framework in Core Foundation
author: Kolin Stürt
tags : [CommonCrypto, ios, Core Foundation, tutorial]
redirect_from:
  - /lessons/2014/01/01/common_crypto
---
{% include JB/setup %}

## AES Encryption using Common Crypto

In the [previous article](https://kolinsturt.github.io/lessons/2013/05/01/hashing_algorithms_in_core_foundation) you leanred about hashing. In this article you'll apply it to encrypting data.

### AES

Here is a demonstration of how to encrypt and decrypt data using the CommonCrypto library in Core Foundation. You'll use a symmetric-key algorithm using the AES specification - AES128. 128 refers to the key size. The AES standard is based on the Rijndeal cipher and uses a substitution-permutation network. It uses a 128 bit block size and operates on a four by four flattened array of bytes in linear memory. AES encrypts data given a key and often you derive the key from a user supplied password. 

### Secure Random Generator

You can adapt this code to work with a password that the user supplies. For now, generate a random password:

	uint8_t password[kCCKeySizeAES128];
    OSStatus result = SecRandomCopyBytes(kSecRandomDefault, kCCKeySizeAES128, password); //  /dev/random is fed by entropy to this function
    if (result != errSecSuccess)
    {
        CFShow(CFSTR("\nCould not create password"));
    }
    
The underlying functionality of the SecRandomCopyBytes function uses the /dev/random Unix file which relies upon the 160-bit Yarrow algorithm. It's based on SHA1 to generate pseudorandom numbers. Depending on the hardware (iOS verses Mac OS), system events, device sensors and interrupt timing during boot generate the entropy.

**NOTE: To look at the open source code of the kernel entropy function, see the [collectEntropy call](https://www.opensource.apple.com/source/securityd/securityd-40600/src/entropy.cpp). The KERN_KDGETENTROPY define in the code calls the [kdbg_getentropy function](https://www.opensource.apple.com/source/xnu/xnu-1456.1.26/bsd/kern/kdebug.c)**

Now that you have a password, you should salt the password. Without a salt, your code would derive the same key when multiple users supply the same password:

	uint8_t salt[8];
    result = SecRandomCopyBytes(kSecRandomDefault, 8, salt);
    if (result != errSecSuccess)
    {
        CFShow(CFSTR("\nCould not create salt"));
    }

This code helps prevent dictionary attacks and rainbow table attacks by adding random data to the password.

### Password-Based Key Derivation

Now that you have a solid password, you'll use it to generate a key to encrypt the data. Instead of making up a key of random characters, it's best practice to use special function, called Password-Based Key Derivation Function 2 - PBKDF2 for short.

PBKDF2 calls a pseudorandom function, combines the salt value repeats hashing operations many times to derive the key. Developers refer to this as key stretching. It expands the time it would take to operate on a set of keys during a brute force attack. PBKDF2 is an improvement over PBKDF1 in that PBKDF1 can only derive keys up to 160 bits in length.

Use the `CCKeyDerivationPBKDF` function with the `kCCPBKDF2` parameter to create an AES key:

    uint8_t derivedKey;
    CCCryptorStatus cryptResult = CCKeyDerivationPBKDF(kCCPBKDF2, (const char *)password, sizeof(password), salt, sizeof(salt), kCCPRFHmacAlgSHA1, 10000, &derivedKey, kCCKeySizeAES128);
    if (cryptResult != kCCSuccess)
    {
        CFShow(CFSTR("\nCould not create key"));
    }
    
In this code, you pass in the function type, `kCCPBKDF2`. Then you provide the password and size of password as well as the salt and size of the salt. The sixth parameter takes a `CCPseudoRandomAlgorithm`. It defines which pseudorandom algorithm to use. You used `kCCPRFHmacAlgSHA1` - PRF stands for the *pseudorandom function* family - the HMAC with the SHA-1 algorithm. Then you pass in how many rounds of the algorithm to apply. The higher the number, the longer it takes to operate on a set of keys during a brute force attack. The last parameters take the output key and length. 

Now that you have a suitable key, package it in a data object:

	CFDataRef keyData = CFDataCreate(kCFAllocatorDefault, &derivedKey, kCCKeySizeAES128);

### Initialization Vectors


The last thing you need to do before encrypting the data is come up with a random initialization vector. This so that the same message does not encrypt to the same cipher text on subsequent operations. Also, the encryption function  computes the IV against the first block of plaintext which then dominoes to the rest of the blocks in the chain. Without an IV, if a message were to start off the same, the output would aid cryptanalysis, especially if they knew the characters of the start of the message. Create an IV now and also add it to a data object:

    uint8_t ivBytesChar[kCCBlockSizeAES128];
    result = SecRandomCopyBytes(kSecRandomDefault, kCCBlockSizeAES128, ivBytesChar);
    if (result != errSecSuccess)
    {
        CFShow(CFSTR("\nCould not create IV"));
    }
    CFDataRef ivData = CFDataCreate(kCFAllocatorDefault, (const UInt8 *)ivBytesChar, kCCBlockSizeAES128);
    
### Encrypting the Data    
    
Now that you have all the necessary parts, encrypt the data using the `CCCrypt` function. Here is it's definition:

	CCCryptorStatus CCCrypt(
	    CCOperation op,         /* kCCEncrypt, etc. */
	    CCAlgorithm alg,        /* kCCAlgorithmAES128, etc. */
	    CCOptions options,      /* kCCOptionPKCS7Padding, etc. */
	    const void *key,
	    size_t keyLength,
	    const void *iv,         /* optional initialization vector */
	    const void *dataIn,     /* optional per op and alg */
	    size_t dataInLength,
	    void *dataOut,          /* data RETURNED here */
	    size_t dataOutAvailable,
	    size_t *dataOutMoved)
	    __OSX_AVAILABLE_STARTING(__MAC_10_4, __IPHONE_2_0);


The first parameter tells it to either encrypt or decrypt the data followed by the algorithm used. The third parameter includes options and the rest of the parameters pass in the key, iv, data, and out data. The options parameter: Currently there are no options for stream ciphers but there are two options for block ciphers: `kCCOptionPKCS7Padding` and `kCCOptionECBMode`. 

ECB stands for Electronic Code Block, passing in this mode means that it divides a message into blocks and each one gets encrypted on its own. Identical blocks would output identical encrypted blocks so that it would be easy to see patters in the encrypted data. It is not recommended to use this mode. Leaving it out of the options means you'll use the default - CBC mode. CBC stands for Cipher-Block Chaining. It means that it XORs each encrypted block with the previous one so that each block depends on all blocks processed up until that block. This is a clear reason why this mode needs an IV. If there's no IV then identical messages with the same key would produce identical sequences.

Another option is `kCCOptionPKCS7Padding`. Because you're using a block cipher, if the message doesn't fit into a multiple of the block size you'll need to add padding to it. PKCS#7 works by padding whole bytes. The value of the added bytes are equal to the number of bytes added. Here is the CCCrypt code:

    //encrypt
    size_t outLength;
    CFIndex encryptedLength = CFDataGetLength(plainTextData) + kCCBlockSizeAES128;
    CFMutableDataRef encryptedData = CFDataCreateMutable(kCFAllocatorDefault, encryptedLength);
    cryptResult = CCCrypt(kCCEncrypt,
                          kCCAlgorithmAES128,
                          kCCOptionPKCS7Padding,
                          CFDataGetBytePtr(keyData),
                          CFDataGetLength(keyData),
                          CFDataGetBytePtr(ivData),
                          CFDataGetBytePtr(plainTextData),
                          CFDataGetLength(plainTextData),
                          CFDataGetMutableBytePtr(encryptedData),
                          encryptedLength,
                          &outLength);
    if (cryptResult == kCCSuccess)
    {
        CFDataSetLength(encryptedData, outLength);
    }
    else
    {
        CFShow(CFSTR("\nEncryption error"));
    }
    
For this example, package all the necessary data and return it so that a user can decrypt the data at a later point:

	CFMutableDictionaryRef encryptionDictionary = CFDictionaryCreateMutable(kCFAllocatorDefault, 3, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
    CFDictionarySetValue(encryptionDictionary, kEncryptedData, encryptedData);
    CFDictionarySetValue(encryptionDictionary, kEncryptionKey, keyData);
    CFDictionarySetValue(encryptionDictionary, kEncryptedInitializationVector, ivData);
    CFRelease(encryptedData);
    CFRelease(keyData);
    CFRelease(ivData);
    
You packaging the encrypted data, key and IV into a dictionary so that you can use it to decrypt the data. Note that you're not storing and should never store the AES key. At the top of the code, add the constants for the dictionary keys:

	const CFStringRef kEncryptedData = CFSTR("EncryptedData");
	const CFStringRef kEncryptionKey = CFSTR("EncryptedKey");
	const CFStringRef kEncryptedInitializationVector = CFSTR("EncryptedIV");

The full function looks like this:

	CFDictionaryRef EncryptAES128FromData(CFDataRef plainTextData)
	{
	    //password
	    uint8_t password[kCCKeySizeAES128];
	    OSStatus result = SecRandomCopyBytes(kSecRandomDefault, kCCKeySizeAES128, password); //  /dev/random is used
	    if (result != errSecSuccess)
	    {
	        CFShow(CFSTR("\nCould not create password"));
	    }
	    
	    //salt
	    uint8_t salt[8];
	    result = SecRandomCopyBytes(kSecRandomDefault, 8, salt);
	    if (result != errSecSuccess)
	    {
	        CFShow(CFSTR("\nCould not create salt"));
	    }
	    
	    //key
	    uint8_t derivedKey;
	    CCCryptorStatus cryptResult = CCKeyDerivationPBKDF(kCCPBKDF2, (const char *)password, sizeof(password), salt, sizeof(salt), kCCPRFHmacAlgSHA1, 10000, &derivedKey, kCCKeySizeAES128);
	    if (cryptResult != kCCSuccess)
	    {
	        CFShow(CFSTR("\nCould not create key"));
	    }
	    CFDataRef keyData = CFDataCreate(kCFAllocatorDefault, &derivedKey, kCCKeySizeAES128);
	    
	    //generate an initialization vector
	    uint8_t ivBytesChar[kCCBlockSizeAES128];
	    result = SecRandomCopyBytes(kSecRandomDefault, kCCBlockSizeAES128, ivBytesChar);
	    if (result != errSecSuccess)
	    {
	        CFShow(CFSTR("\nCould not create IV"));
	    }
	    CFDataRef ivData = CFDataCreate(kCFAllocatorDefault, (const UInt8 *)ivBytesChar, kCCBlockSizeAES128);
	    
	    //encrypt
	    size_t outLength;
	    CFIndex encryptedLength = CFDataGetLength(plainTextData) + kCCBlockSizeAES128;
	    CFMutableDataRef encryptedData = CFDataCreateMutable(kCFAllocatorDefault, encryptedLength);
	    cryptResult = CCCrypt(kCCEncrypt,
	                          kCCAlgorithmAES128,
	                          kCCOptionPKCS7Padding,
	                          CFDataGetBytePtr(keyData),
	                          CFDataGetLength(keyData),
	                          CFDataGetBytePtr(ivData),
	                          CFDataGetBytePtr(plainTextData),
	                          CFDataGetLength(plainTextData),
	                          CFDataGetMutableBytePtr(encryptedData),
	                          encryptedLength,
	                          &outLength);
	    if (cryptResult == kCCSuccess)
	    {
	        CFDataSetLength(encryptedData, outLength);
	    }
	    else
	    {
	        CFShow(CFSTR("\nEncryption error"));
	    }
	    
	
	    CFMutableDictionaryRef encryptionDictionary = CFDictionaryCreateMutable(kCFAllocatorDefault, 3, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
	    CFDictionarySetValue(encryptionDictionary, kEncryptedData, encryptedData);
	    CFDictionarySetValue(encryptionDictionary, kEncryptionKey, keyData);
	    CFDictionarySetValue(encryptionDictionary, kEncryptedInitializationVector, ivData);
	    CFRelease(encryptedData);
	    CFRelease(keyData);
	    CFRelease(ivData);
	    return CFAutorelease(encryptionDictionary);
	}
	
Now that you've encrypted the data. You'll need to write code to decrypt it.
	
### Decrypting the Data
	
To decrypt the data, you use the same `CCCrypt` function. It has exactly the same setup except you pass `kCCDecrypt` as the first parameter:

	CFDataRef DecryptAES128File(CFDictionaryRef cipherText)
	{
	    //get encrypted data as data object
	    CFDataRef encryptedData = CFDictionaryGetValue(cipherText, kEncryptedData);
	    
	    //get the key
	    CFDataRef key = CFDictionaryGetValue(cipherText, kEncryptionKey);
	    
	    //get the initialization vector
	    CFDataRef iv = CFDictionaryGetValue(cipherText, kEncryptedInitializationVector);
	    
	    size_t outLength = 0;
	    size_t decryptedLength = CFDataGetLength(encryptedData) + kCCBlockSizeAES128;
	    CFMutableDataRef decryptedData = CFDataCreateMutable(kCFAllocatorDefault, decryptedLength);
	    CCCryptorStatus cryptResult = CCCrypt(kCCDecrypt,
	                          kCCAlgorithmAES128,
	                          kCCOptionPKCS7Padding,
	                          CFDataGetBytePtr(key),
	                          CFDataGetLength(key),
	                          CFDataGetBytePtr(iv),
	                          CFDataGetBytePtr(encryptedData),
	                          CFDataGetLength(encryptedData),
	                          CFDataGetMutableBytePtr(decryptedData),
	                          decryptedLength,
	                          &outLength);
	    if (cryptResult == kCCSuccess)
	    {
	        CFDataSetLength(decryptedData, outLength);
	    }
	    else
    	{
        	CFShow(CFSTR("\nDecryption error"));
    	}
	
	    return CFAutorelease(decryptedData);
	}
	
Now that you have both encrypt and decrypt functions, it's time to test the code.

### Putting It All Together

To test and implement the functions, CFDictionaryRef and CFDataRef are toll-free bridged with their NSDictionary* and NSData* counterparts. Here is how you might use this in an iOS XCode project:

	NSDictionary *encryptedDictionary = (__bridge NSDictionary *)EncryptAES128FromData((__bridge CFDataRef)[@"test string to encrypt" dataUsingEncoding:NSUTF8StringEncoding]);
    
    NSData *data = (__bridge NSData *)DecryptAES128File((__bridge CFDictionaryRef)encryptedDictionary);
    
    NSString *decodedString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    
    NSLog(@"Decrypted string is: %@", decodedString);

While this test outputs the decrypted string to confirm it works, remember not to log passwords and keys to the console in production. The console log is easily retrieved. In production projects, it's better to set breakpoints and inspect variables for security-related code instead of logging. That way your team won't forget to remove them when releasing the code.

As always, the full source code for this tutorial can be found [here](https://github.com/CollinBStuart/AESEncryption).

If you're intending to do AES exclusively on a specific mobile platform, check out the [iOS Encryption Tutorial](http://code.tutsplus.com/tutorials/securing-ios-data-at-rest-encryption--cms-28786) and [Encryption Tutorial for Android](https://www.raywenderlich.com/778533-encryption-tutorial-for-android-getting-started).

If you're interested in learning more about other popular modes of operation, check out [AES GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode).
