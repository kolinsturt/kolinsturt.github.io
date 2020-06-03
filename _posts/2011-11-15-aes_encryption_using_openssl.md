---
layout: post
category : lessons
tagline: "OpenSSL AES Encryption"
title: AES encryption using OpenSSL
description: Implementing AES encryption in OpenSSL
author: Kolin Stürt
tags : [C++, OpenSSL, AES, Encryption, CBC, PBKDF2]
---
{% include JB/setup %}

## AES Encryption using OpenSSL

AES is a fast block cipher and symmetric encryption standard. It encrypts data using a key. You should always salt, or stretch, a key. A simple string or password is not enough entropy. Instead you'll derive a key from a user-supplied password by applying random data, called a salt, and then hash it several times over. You'll use a salt so that if a person uses the same password elsewhere, the output will always be different.

In the [previous article](https://kolinsturt.github.io/lessons/2013/04/01/sha_using_openssl) you leanred about hashing. If that is new to you, check out that article first.

### Applying Password-Based Key Derivation

You can implement this all yourself, however a key derivation function has already been developed for this purpose – called PBKDF2 (Password-Based Key Derivation Function 2). It performs the functions many times over to stretch the key and help prevent password cracking.

Here is some code to create a 256 bit key using OpenSSL's `PKCS5_PBKDF2_HMAC` function. It's parameters are the following:
1. The password.
2. Length of the password.
3. The salt.
4. Length of the salt.
5. Number of iterations to perform.
6. The specific hash function.
7. Length of the key.
8. The out data.

	    int iterations = 1000;
	    size_t keyLength  = 32; //256 bits
	    size_t i;
	    unsigned char *keyChar;
	    const char passwordChar[] = "qf4i2btz";
	    unsigned char saltChar[] = {'q','j','6','-'};
	    keyChar = (unsigned char *)malloc(sizeof(unsigned char) * keyLength);
	    cout << "password is " << passwordChar << endl;
	    cout << "salt is ";
	    for (i = 0; i < sizeof(saltChar); i++)
	    {
	        printf("%02x", saltChar[i]);
	    }
	    cout << endl;
	    if (PKCS5_PBKDF2_HMAC(passwordChar, strlen(passwordChar), saltChar, sizeof(saltChar), iterations, EVP_sha512(), keyLength, keyChar) != 0 )
	    {
	        cout << "key is ";
	        for (i = 0; i < keyLength; i++)
	        {
	            printf("%02x", keyChar[i]);
	        }
	        cout << endl;
	    }
	    else
	    {
	        cout << "Failure to create key for password - PKCS5_PBKDF2_HMAC_SHA1" << endl;
	    }
	    
	    // ...
	    
	    free(keyChar);

This will always give the same result given the same iterations, salt and password. So if a user enters their password, you would perform this function to get the correct key.

### Initialization Vectors

There are different modes of encryption - You'll use CBC mode. The reason for this is that in CBC mode each next plain text block is XOR'd with the previous cipher text block which makes for stronger encryption. The problem with CBC is that the first block is not as unique. If there are common words in the first block, then you could perform a dictionary attack on it. To get around this, you'll use an initialization vector (IV) of random bytes that are XOR'd with the first block.

Here is a 128 bit IV:

	unsigned char *iv = (unsigned char *)"73472859478267948";

Create the above with a good secure random number generator for your platform. One option is [arc4random](https://security.stackexchange.com/questions/85601/is-arc4random-secure-enough). Another is [SecRandomCopyBytes](https://developer.apple.com/documentation/security/1399291-secrandomcopybytes).

Although the IV is considered public, avoid using constant or sequential IV's. Do not reuse an IV for more than one message if you are using the same key.

### Encrypting The Data

To begin, create a context and initialize it:

	EVP_CIPHER_CTX *cipherContext;
	cipherContext = EVP_CIPHER_CTX_new();
	EVP_CIPHER_CTX_init(cipherContext);

Next set up the type of encryption operation you're are doing, in this case AES. Lookup the correct values for the standard. AES-256 uses a 256 bit key, and 128 bit IV size (AES256 refers to the key size. The block and IV size are still [128 bits](http://csrc.nist.gov/publications/fips/fips197/fips-197.pdf)). The following [function](https://www.openssl.org/docs/crypto/EVP_EncryptInit.html) performs key setup:

	EVP_EncryptInit_ex(cipherContext, EVP_aes_256_cbc(), NULL, keyChar, ivChar);

Use the update function to provide cleartext data. You can call it multiple times to append unencrypted data.

	EVP_EncryptUpdate(cipherContext, cipherTextChar, &length, plainTextChar, plainTextLength);

When you're finished adding data, call the finalize function. This will finish writing all encrypted bytes:

	EVP_EncryptFinal_ex(cipherContext, cipherTextChar + length, &length);

The [EVP](https://www.openssl.org/docs/crypto/evp.html) methods are high level functions of the OpenSSL library. While you can access the same functionality using raw lower level parts of the API, it is not recommended as it may introduce more security holes if you are not careful with the implementation. The previous functions will return 1 for success and 0 for failure.

Here is the full function:

	#include <openssl/evp.h>
	#include <openssl/conf.h>
	#include <openssl/err.h>
	#include <openssl/bio.h>
	#include <openssl/x509.h>
	#include <openssl/hmac.h>
	
	int Crypto::_encrypt(unsigned char *plainTextChar, int plainTextLength, unsigned char *keyChar, unsigned char *ivChar, unsigned char *cipherTextChar)
	{
	    EVP_CIPHER_CTX *cipherContext;
	    int length;
	    int cipherTextLength;
	    
	    if ( ! (cipherContext = EVP_CIPHER_CTX_new()) )
	    {
	        _handleErrors();
	    }
	    EVP_CIPHER_CTX_init(cipherContext);
	    
	    if (1 != EVP_EncryptInit_ex(cipherContext, EVP_aes_256_cbc(), NULL, keyChar, ivChar) )
	    {
	        _handleErrors();
	    }
	    
	    if (1 != EVP_EncryptUpdate(cipherContext, cipherTextChar, &length, plainTextChar, plainTextLength))
	    {
	        _handleErrors();
	    }
	    
	    cipherTextLength = length;
	
	    if (1 != EVP_EncryptFinal_ex(cipherContext, cipherTextChar + length, &length))
	    {
	        _handleErrors();
	    }
	    
	    cipherTextLength += length;
	    
	    EVP_CIPHER_CTX_cleanup(cipherContext);
	    EVP_CIPHER_CTX_free(cipherContext);
	    
	    return cipherTextLength;
	}

Padding is used by default if the data does not divide exactly into the fixed block size.

### Decrypting The Data

To decrypt the data, create and initialize the context with a decryption operation using the same key size:

	cipherContext = EVP_CIPHER_CTX_new();
	EVP_DecryptInit_ex(cipherContext, EVP_aes_256_cbc(), NULL, keyChar, ivChar);
	EVP_CIPHER_CTX_init(cipherContext);

As before, you can call the decrypted operation on multiple input sources:

	EVP_DecryptUpdate(cipherContext, plainTextChar, &length, cipherTextChar, cipherTextLength);

Call this to write the final plaintext bytes:

	EVP_DecryptFinal_ex(cipherContext, plainTextChar + length, &length);

Here is the full decrypt function:

	int Crypto::_decrypt(unsigned char *cipherTextChar, int cipherTextLength, unsigned char *keyChar, unsigned char *ivChar, unsigned char *plainTextChar)
	{
	    EVP_CIPHER_CTX *cipherContext;
	    int length;
	    int plainTextLength;
	    
	    if ( ! (cipherContext = EVP_CIPHER_CTX_new()) )
	    {
	        _handleErrors();
	    }
	    EVP_CIPHER_CTX_init(cipherContext);
	    
	    if (1 != EVP_DecryptInit_ex(cipherContext, EVP_aes_256_cbc(), NULL, keyChar, ivChar))
	    {
	        _handleErrors();
	    }
	
	    if (1 != EVP_DecryptUpdate(cipherContext, plainTextChar, &length, cipherTextChar, cipherTextLength))
	    {
	        _handleErrors();
	    }
	    plainTextLength = length;
	    
	 
	    if (1 != EVP_DecryptFinal_ex(cipherContext, plainTextChar + length, &length))
	    {
	        _handleErrors();
	    }
	    plainTextLength += length;
	    
	    EVP_CIPHER_CTX_cleanup(cipherContext);
	    EVP_CIPHER_CTX_free(cipherContext);
	    
	    return plainTextLength;
	}
	
Here is the _handleErrors() function used in the above code:

	void Crypto::_handleErrors(void)
	{
	    ERR_print_errors_fp(stderr);
	    abort();
	}
	
### Putting it All Together

There are a few more things you should know about the functions:

- To configure OpenSSL using a configuration file, call `OPENSSL_config()`. You can pass `NULL` to use the system default *.cnf* configuration file. To free up configuration resources, call `CONF_modules_free()`.
- For debugging purposes, if you would like to print out error messages as strings, you must call the `ERR_load_crypto_strings()` function first. Call `ERR_free_strings()` to clean up the resources and free memory. Remember to remove debug logs that output sensitive strings or the key to the console!
- OpenSSL contains a table of available algorithms which is used when calling functions such as `EVP_get_cipher_byname()`. It's also used internally so that if not initialized, some functions will fail. To add algorithms to the internal table, call `OpenSSL_add_all_algorithms()`. When you finish or before your application exits, call `EVP_cleanup()` which removes all algorithms from the table.
- Once you have encrypted the plaintext, you can store the encrypted binary data. If you need to send or store it in a text format such as XML or JSON, you'll need to encode the binary information. For example, you can use [Base64](https://en.wikipedia.org/wiki/Base64) to encode the encrypted binary data. For a Base64 utility, see [here](https://github.com/CollinStuart/Base64CPP).

Make a test function by wrapping the C code in a higher level C++ class, call it *Crypto*. You could additionally pass `std::string` in and out of the class if you want. The following example adds the extra convenience of encoding and decoding the binary data to and from Base64. If you don't feel like adding a Base64 utility to your code base, you can comment out that optional step below:

	void Crypto::encryptTest()
	{
	    //Can be changed to generate a key from the given password
	    int iterations = 1000;
	    size_t keyLength  = 32; //256 bits
	    size_t i;
	    unsigned char *keyChar;
	    const char passwordChar[] = "qf4i2btz";
	    unsigned char saltChar[] = {'q','j','6','-'};
	    keyChar = (unsigned char *)malloc(sizeof(unsigned char) * keyLength);
	    //Remove the following in production code:
	    cout << "password is " << passwordChar << endl;
	    cout << "salt is ";
	    for (i = 0; i < sizeof(saltChar); i++)
	    {
	        printf("%02x", saltChar[i]);
	    }
	    cout << endl;
	    if (PKCS5_PBKDF2_HMAC(passwordChar, strlen(passwordChar), saltChar, sizeof(saltChar), iterations, EVP_sha512(), keyLength, keyChar) != 0 )
	    {
	        //Comment out for production code:
	        cout << "key is ";
	        for (i = 0; i < keyLength; i++)
	        {
	            printf("%02x", keyChar[i]);
	        }
	        cout << endl;
	    }
	    else
	    {
	        cout << "Failure to create key for password - PKCS5_PBKDF2_HMAC_SHA1" << endl;
	    }
	
	    unsigned char *ivChar = (unsigned char *)"73472859478267948"; //128 bit IV - use a secure generator
	    unsigned char *plainTextChar = (unsigned char *)"Secret text to be encrypted";
	
	    //Test buffer for cipher text. For this test we must make sure this is long enough for the
	    //encrypted data. It could end up being longer than plain text
	    unsigned char cipherTextChar[128];
	
	    unsigned char decryptedTextChar[128];
	
	    int decryptedTextLength, cipherTextLength;
	
	    //Init OpenSSL library
	    ERR_load_crypto_strings();
	    OpenSSL_add_all_algorithms();
	    OPENSSL_config(NULL);
	
	    //do the actual encryption
	    cipherTextLength = _encrypt(plainTextChar, strlen((const char *)plainTextChar), keyChar, ivChar, cipherTextChar);
	
	    
	    
	    
	    //-----
	    //Optional example, this part can be skipped. Convert binary data to Base64 string
	    //If your storing the data as binary, then you can pass in binary data to be decrypted
	    std::string base64EncryptedString = Base64::base64Encode(cipherTextChar, cipherTextLength);
	    cout << "Cipher text is " << base64EncryptedString << endl;
	    
	    //Now convert the Base64 string back to data
	    unsigned char encryptedDataChar[128];
	    Base64::base64DecodeToData(base64EncryptedString, encryptedDataChar, cipherTextLength);
	    //-----
	    
	    
	    
	    
	
	    //do the actual decryption
	    decryptedTextLength = _decrypt(cipherTextChar, cipherTextLength, keyChar, ivChar, decryptedTextChar);
	
	    //We know we passed in text at the beginning, so add a NULL terminator to be compliant with printable text
	    decryptedTextChar[decryptedTextLength] = '\0';
	
	    //wrap it into something useful or higher level...
	    std::string decryptedString((char *)decryptedTextChar, decryptedTextLength);
	    cout << "Decrypted text is:" << decryptedString << endl;
	    
	    //cleanup
	    EVP_cleanup();
	    ERR_free_strings();
	    CONF_modules_free();
	    free(keyChar);
	}

While this example knows the length of the plain text and output, you can figure out the length of the output by ( (inputLength / blockSize) * blockSize) + blockSize.

Remember to remove all console logging for production code.

That's it for implementing AES 256 CBC in OpenSSL. To learn how to perform AES encryption using Apple’s CommonCrypto library see [this article](https://kolinsturt.github.io/lessons/2014/01/01/common_crypto). If you're interested in learning more about other popular modes of operation, check out [AES GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode).
