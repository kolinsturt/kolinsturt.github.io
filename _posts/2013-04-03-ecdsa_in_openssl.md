---
layout: post
category : lessons
tagline: "OpenSSL ECDSA"
title: ECDSA in OpenSSL
description: ECDSA with OpenSSL in C++
author: Kolin Stürt
tags : [OpenSSL, C++, hash, SHA, message digest, ecdsa, ec, Elliptic Curve Cryptography]
---
{% include JB/setup %}

## ECDSA in OpenSSL

In the [previous article](https://kolinsturt.github.io/lessons/2013/04/02/hmac_in_openssl), you learned how to sign data using a symmetric key via HMAC. The problem with symmetric keys are how to exchange the key security. In this article, you'll use Public Key Cryptography to solve that problem. 

Elliptic Curve Cryptography - ECC, is a modern set of algorithms based on elliptic curves over finite fields. ECC keys are smaller in size and faster to generate than other standards such as RSA. A key of 256-bits offers a very strong level of security. You use the private key for signing and the public key for verifying. That way, you can exchange the public key without worrying about compromising the private key.

To get started, create an elliptic curve key-pair. You can sign strings or files that your application parses using the elliptic curve keypair. Make sure you store your private key in a place that never traverses the network. Here are the OpenSSL commands to create signatures and work with the keypair:

#### Generating a Private Key:

 sudo openssl ecparam -name secp256k1 -genkey -noout -out secp256k1-PrivateKey.pem

#### Generating the Corresponding Public Key:

sudo openssl ec -in secp256k1-PrivateKey.pem -pubout -out secp256k1-PublicKey.pem

#### Creating a Signature:

  openssl dgst -sha1 -sign private.pem fileToSign.xml > signature.bin

* Older OpenSSL versions use `-ecdsa-with-SHA1` in place of the `-sha` parameter.

#### Base64 Encoding the Signature:

  openssl enc -base64 -in signature.bin -out signature.txt

* Remove line breaks for compatibility across platforms.

#### Verifying Using the Command Line (optional)

First Base64 decode the signature:

  openssl enc -base64 -d -in signature.txt -out signatureDec.bin

Then verify the binary signature:

  openssl dgst -ecdsa-with-SHA1 -verify public.pem -signature signatureDec.bin test.xml
  

#### Verifying in Your App

Unlike RSA, you do not need to hash the data prior to signing with ECDSA. You'll support verifying both binary and text files with an `isBinary` flag. It's redundant to open the file to verify and then open it again to read it's contence once verified. You'll add a `fileData` variable to pass in to the function that returns the bytes read if verified. Here's an example implementation in OpenSSL:

    bool InputValidator::verifyFile(const std::string filenameString, const std::string signatureString, const bool isBinary, std::vector<u_int8_t> *fileData)
    {
        bool success = false;
        if ( ! filenameString.empty() && ! signatureString.empty())
        {
            //1. You're public key here
            const char *pubKey = "-----BEGIN PUBLIC KEY-----\r\nMFYwEAYH....waxyPQfdKUZA==\r\n-----END PUBLIC KEY-----";
            if (pubKey)
            {
                size_t len = strlen(pubKey);
                BIO *bio = BIO_new(BIO_s_mem()); //2
                EVP_PKEY *evpKey = NULL;
                if (BIO_write(bio, pubKey, (int)len) == (int)len) //3
                {
                    evpKey = PEM_read_bio_PUBKEY(bio, NULL, NULL, NULL); //4
                }
                BIO_free_all(bio); //5
                
                if (evpKey)
                {
                    unsigned char *signatureChars = NULL;
                    size_t signatureLength;
                    if (Encoding::newBase64Decode((char *)signatureString.c_str(), &signatureChars, &signatureLength))
                    {
                        EVP_MD_CTX messageDigestContext;
                        EVP_MD_CTX_init(&messageDigestContext); //This is here for consistency - in case the implementation chages
                        EVP_VerifyInit(&messageDigestContext, EVP_ecdsa()); //6
                        
                        if (isBinary) //Support verifying both binary and text files
                        {
                            std::ifstream inputFileStream(filenameString, std::ios::binary); //Open the file as binary.
                            if (fileData) //Return the bytes read if verified.
                            {
                                *fileData = std::vector<unsigned char>((std::istreambuf_iterator<char>(inputFileStream)),(std::istreambuf_iterator<char>()));
                                EVP_VerifyUpdate(&messageDigestContext, fileData->data(), fileData->size()); //7
                            }
                            else
                            {
                                std::vector<unsigned char> contentVector((std::istreambuf_iterator<char>(inputFileStream)),(std::istreambuf_iterator<char>()));
                                EVP_VerifyUpdate(&messageDigestContext, contentVector.data(), contentVector.size());
                            }
                        }
                        else //Instead, read as char
                        {
                            std::ifstream inputFileStream(filenameString);
                            std::string contentsString((std::istreambuf_iterator<char>(inputFileStream)), std::istreambuf_iterator<char>());
                            const char *contentChars = contentsString.c_str();
                            EVP_VerifyUpdate(&messageDigestContext, contentChars, strlen(contentChars));
                        }
                        
                        int rc = EVP_VerifyFinal(&messageDigestContext, signatureChars, static_cast<unsigned int>(signatureLength), evpKey); //8
                        if (rc == 1)
                        {
                            success = true;
                        }
                        else
                        {
                            //Don't return unverified data. Note data was first set before verified. Another option is to only set the data after verification is successful.
                            *fileData = std::vector<unsigned char>();
                        }
                        
                        //cleanup
                        if (signatureChars)
                        {
                            free(signatureChars);
                        }
                        EVP_MD_CTX_cleanup(&messageDigestContext);
                    }
                    
                    EVP_PKEY_free(evpKey);
                }
            } //end if (pubKey)
        } //end if ( ! filenameString.empty() && ! signatureString.empty())
        
        return success;
    }
    
1. Add you're public key here.
2. Create a memory Basic Input/Output Stream.
3. Write the data to the `BIO`.
4. Feed the `BIO` into the PEM read function to create an EVP Public Key Object.
5. Because you used "new", free the bio object.
6. Initialize a verification context with the digest type ECDSA.
7. You hash `size()` bytes from `data()` into the verification context. While you can call this multiple times, you're adding all the data at once. You're using this way instead of one-shot functions in case the implementation changes and you later need to add data from a stream, for example.
8. `EVP_VerifyFinal` finalizes a copy of the data in the digest context. That means you can still call `EVP_VerifyUpdate` and `EVP_VerifyFinal` later to digest and verify more data.
 
This example doesn't return unverified data. Be careful with this style where the data is set earlier. As long as their's no returns or exceptions an attacker won't be able to change the flow to allow for unverified data. Another style is to only set the data after verification was successful.
    
That concludes the series on Crypto in C++. If you're looking to implement this on mobile platforms, check out the [Digital Signatures With Swift](http://code.tutsplus.com/tutorials/creating-digital-signatures-with-swift--cms-29287?_ga=2.107394370.151438550.1591542137-2011297255.1591542137) and [Authentication on Android Section](https://www.raywenderlich.com/10056112-securing-network-data-tutorial-for-android#toc-anchor-010).
