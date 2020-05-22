---
layout: post
category : lessons
tagline: "Using the CommonCrypto library in CoreFoundation"
title: Encryption on iOS using Common Crypto
description: Encryption using the Common Crypto framework
author: Kolin Stürt
tags : [CommonCrypto, ios, Core Foundation, tutorial]
---
{% include JB/setup %}

## AES Encryption - Using the CommonCrypto Library in Core Foundation

### Secure Random Generator

Here is a demonstration of how to encrypt and decrypt data using the CommonCrypto library. We will be using a symmetric-key algorithm using the AES specification - AES128 (128 refers to the key size that is used). The AES standard is based on the Rijndeal cipher and uses a substitution-permutation network. It uses a 128 bit block size and operates on a four by four flattened array of bytes in linear memory. AES encrypts data given a key and often the key is derived from a user inputted password. You may have a place in your application where the user can enter a password. We will cover the security of passwords in another tutorial. The function we will create can be adapted to work with a password that the user enters, but for our example we will just generate a random password each time. Lets do that now

	uint8_t password[kCCKeySizeAES128];
    OSStatus result = SecRandomCopyBytes(kSecRandomDefault, kCCKeySizeAES128, password); //  /dev/random is fed by entropy to this function
    if (result != errSecSuccess)
    {
        CFShow(CFSTR("\nCould not create password"));
    }
    
The underlying functionality of the SecRandomCopyBytes function uses the /dev/random Unix file which relies upon the 160-bit Yarrow algorithm, based on SHA1 to generate pseudorandom numbers. Depending on the hardware (iOS verses Mac OS), generally the entropy is retrieved from system events, device sensors, as well as interrupt timing during boot.

**NOTE: To look at the open source code of the kernel entropy function, see the collectEntropy call [here](https://www.opensource.apple.com/source/securityd/securityd-40600/src/entropy.cpp). The KERN_KDGETENTROPY define in the code calls the kdbg_getentropy function [here](https://www.opensource.apple.com/source/xnu/xnu-1456.1.26/bsd/kern/kdebug.c)**

Now we have our password. However, if you are going to adapt this code so that instead a user enters a password, in addition to strong password checking, it is a good idea to salt the password. This will help prevent dictionary attacks and rainbow table attacks by adding additional random data to the password.

	uint8_t salt[8];
    result = SecRandomCopyBytes(kSecRandomDefault, 8, salt);
    if (result != errSecSuccess)
    {
        CFShow(CFSTR("\nCould not create salt"));
    }

### Password-Based Key Derivation

Now that we have a more solid password, we will use it to generate a key that will be used to encrypt our data. Instead of just making up a key of random numbers, we are going to use a special function, called Password-Based Key Derivation Function 2, or PBKDF2 for short. The reason for this is that in addition to the function also calling a pseudorandom function, it combines the salt value as well as repeating operations many times to finally derive the key. This further secures the key, often referred to as key stretching, in expanding the time it would take to operate on a set of keys during a brute force attack. PBKDF2 is an improvement over PBKDF1 in that PBKDF1 can only derive keys up to 160 bits in length.

    uint8_t derivedKey;
    CCCryptorStatus cryptResult = CCKeyDerivationPBKDF(kCCPBKDF2, (const char *)password, sizeof(password), salt, sizeof(salt), kCCPRFHmacAlgSHA1, 10000, &derivedKey, kCCKeySizeAES128);
    if (cryptResult != kCCSuccess)
    {
        CFShow(CFSTR("\nCould not create key"));
    }
    
As you can see in this function, we first pass in the function type. Right now we only have kCCPBKDF2 but as improvements are made there may be more options. Then we pass the password and size of password, as well as the salt and size of the salt. The sixth parameter takes a CCPseudoRandomAlgorithm which defines which pseudorandom algorithm we are going to use, in this case kCCPRFHmacAlgSHA1 (PRF stands for pseudorandom function family. We are using HAMC with the SHA-1 algorithm). Then we pass in how many rounds of the random algorithm we will apply. The last parameters take the output key and length. 

**NOTE: It's not a good idea to log passwords and keys to the console as the console log is easily retrieved in production environments. If there are such logs, remember to remove them in production code.**  

Now we have a suitable key, lets package it in a data object

	CFDataRef keyData = CFDataCreate(kCFAllocatorDefault, &derivedKey, kCCKeySizeAES128);

### Initialization Vectors

The last thing we need to do before we actually encrypt the data is come up with a random initialization vector. We need this so that the same message does not encrypt to the same cipher text on subsequent operations. If a message were to start off the same, without a new random IV to be used on the first block of data, the output could aid cryptanalysis, especially if they knew the characters of the start of the message. The IV is computed against the first block of plaintext which then dominoes to the rest of the blocks in the chain. We will create an IV now and also add it to a data object

    uint8_t ivBytesChar[kCCBlockSizeAES128];
    result = SecRandomCopyBytes(kSecRandomDefault, kCCBlockSizeAES128, ivBytesChar);
    if (result != errSecSuccess)
    {
        CFShow(CFSTR("\nCould not create IV"));
    }
    CFDataRef ivData = CFDataCreate(kCFAllocatorDefault, (const UInt8 *)ivBytesChar, kCCBlockSizeAES128);
    
### Encrypting the Data    
    
Now that we have all the necessary parts, we can go aged and encrypt the data using the CCCrypt function. Here is it's definition:

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

The first parameter tells it to either encrypt or decrypt the data followed by the algorithm used. The third parameter includes options and the rest of the parameters pass in the key, iv, data, and out data. Lets talk about the options parameter. Currently there are no options for stream ciphers but there are two options for block ciphers: kCCOptionPKCS7Padding and kCCOptionECBMode. 

ECB stands for Electronic Code Block, passing in this mode means that the basic system is used where a message is divided into blocks and each one gets encrypted on its own. Identical blocks would output identical encrypted blocks so that it would be easy to see patters in the encrypted data. It is not recommended to use this mode, as leaving it out of the options means the default CBC mode is used. CBC, Cipher-block chaining means that each block that is encrypted is XORed with the previous one so that each block depends on all blocks processed up until that block. This is a clear reason why this mode needs an IV, because if not, identical messages with the same key would produce identical sequences.

Another option we have is kCCOptionPKCS7Padding. Because we are using a block cipher, if the message doesn't fit nicely into a multiple of the block size, we will need to add padding to it. PKCS#7 works by padding whole bytes. The value of the added bytes are equal to the number of bytes that need to be added. Here is the CCCrypt code

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
    
For our example, we will package all the necessary data and return it so that a user can decrypt the data at a later point. This is merely meant to be used as a code example.

	CFMutableDictionaryRef encryptionDictionary = CFDictionaryCreateMutable(kCFAllocatorDefault, 3, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
    CFDictionarySetValue(encryptionDictionary, kEncryptedData, encryptedData);
    CFDictionarySetValue(encryptionDictionary, kEncryptionKey, keyData);
    CFDictionarySetValue(encryptionDictionary, kEncryptedInitializationVector, ivData);
    CFRelease(encryptedData);
    CFRelease(keyData);
    CFRelease(ivData);
    
We are packaging the encrypted data, key and IV into a dictionary so that we can use it in our example to decrypt the data. At the top of our code we can add the constants used for the keys for the dictionary

	const CFStringRef kEncryptedData = CFSTR("EncryptedData");
	const CFStringRef kEncryptionKey = CFSTR("EncryptedKey");
	const CFStringRef kEncryptedInitializationVector = CFSTR("EncryptedIV");

The full function will look like this

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
	
### Decrypting the Data
	
Next, lets write the decrypt function. The CCCrypt function has exactly the same setup except we pass kCCDecrypt as the first parameter.

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

### Putting It All Together

That's the full code for encrypting and decrypting data. This code can of course be adapted to Foundation if you have included Foundation in your project as CFDictionaryRef and CFDataRef are toll-free bridged with their NSDictionary* and NSData* counterparts. Additionally, here is how one might use this in an iOS XCode project:

	NSDictionary *encryptedDictionary = (__bridge NSDictionary *)EncryptAES128FromData((__bridge CFDataRef)[@"test string to encrypt" dataUsingEncoding:NSUTF8StringEncoding]);
    
    NSData *data = (__bridge NSData *)DecryptAES128File((__bridge CFDictionaryRef)encryptedDictionary);
    
    NSString *decodedString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    
    NSLog(@"Decrypted string is: %@", decodedString);



As always, the full source code for this tutorial can be found [here](https://github.com/CollinBStuart/AESEncryption).
