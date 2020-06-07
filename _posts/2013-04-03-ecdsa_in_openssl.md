2013-04-02-hmac_in_openssl.md

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

In the previous article, you learned how to sign data using a symmetric key via HMAC. The problem with symmetric keys are how to exchange the key security. In this article, you'll use Public Key Cryptography to solve that problem. 

Elliptic Curve Cryptography - ECC, is a modern set of algorithms based on elliptic curves over finite fields. ECC keys are smaller in size and faster to generate than other standards such as RSA. A key of 256-bits offers a very strong level of security. You use the private key for signing and the public key for verifying. That way, you can exchange the public key without worrying about compromising the private key.

Unlike RSA, you do not need to hash the data prior to signing with ECDSA. Here's an example implementation in OpenSSL:

    bool InputValidator::verifyFile(const std::string filenameString, const std::string signatureString, const bool isBinary, std::vector<u_int8_t> *fileData)
    {
        bool success = false;
        if ( ! filenameString.empty() && ! signatureString.empty())
        {
            //You're public key
            const char *pubKey = "-----BEGIN PUBLIC KEY-----\r\nMFYwEAYHKoZIzj0CAQYFK4EEAAoDQgAEN4O9oLptRdzyly5mjNJKojdKxaJV9PuM\r\nq6pDzYAU9ZtzOjsjl9csF+jsUG/dV0xQC2GwDe/4qvwaxyPQfdKUZA==\r\n-----END PUBLIC KEY-----";
            if (pubKey)
            {
                size_t len = strlen(pubKey);
                BIO *bio = BIO_new(BIO_s_mem()); //Create a memory Basic Input/Output Stream
                EVP_PKEY *evpKey = NULL;
                if (BIO_write(bio, pubKey, (int)len) == (int)len) //Write the data to the BIO
                {
                    evpKey = PEM_read_bio_PUBKEY(bio, NULL, NULL, NULL); //Feed to BIO into the PEM read function to create an EVP Public Key Object
                }
                BIO_free_all(bio); //Because you used "new", free the bio object
                
                if (evpKey)
                {
                    unsigned char *signatureChars = NULL;
                    size_t signatureLength;
                    if (Encoding::newBase64Decode((char *)signatureString.c_str(), &signatureChars, &signatureLength))
                    {
                        EVP_MD_CTX messageDigestContext;
                        EVP_MD_CTX_init(&messageDigestContext); //Here for consistency reminder - if implementation changes
                        EVP_VerifyInit(&messageDigestContext, EVP_ecdsa()); //Initialize a verification context with the digest type, ECDSA
                        
                        if (isBinary) //You'll support verifying both binary and text files
                        {
                            std::ifstream inputFileStream(filenameString, std::ios::binary); //You open the file as binary.
                            if (fileData) //It's redundant to open the file to verify and then open it again to read it's contence once verified. You added an example of returning the bytes read if verified.
                            {
                                *fileData = std::vector<unsigned char>((std::istreambuf_iterator<char>(inputFileStream)),(std::istreambuf_iterator<char>()));
                                EVP_VerifyUpdate(&messageDigestContext, fileData->data(), fileData->size()); //You hash ->size() bytes from ->data() into the verification context. While you can call this multiple times. You're adding all the data at once. We're using this way instead of one-shot functions in case the implementation changes and you later need to add data from a stream, for example.
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
                        
                        //EVP_VerifyFinal() finalizes a copy of the data in the digest context. That means you can still call EVP_VerifyUpdate() and EVP_VerifyFinal() later to digest and verify more data.
                        int rc = EVP_VerifyFinal(&messageDigestContext, signatureChars, static_cast<unsigned int>(signatureLength), evpKey);
                        if (rc == 1)
                        {
                            success = true;
                        }
                        else
                        {
                            //Don't return unverified data. Be careful with this style. Data was set earlier. As long as their's no returns an attacker won't be able
                            //to change the flow to allow for unverified data. Another option is to only set the data after verification was successful.
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
    
That concludes the serious on crypo in C++. If you're looking to implement this on mobile platforms, check out the [Digital Signatures With Swift](http://code.tutsplus.com/tutorials/creating-digital-signatures-with-swift--cms-29287?_ga=2.107394370.151438550.1591542137-2011297255.1591542137) and [Securing Network Data Tutorial for Android](https://www.raywenderlich.com/10056112-securing-network-data-tutorial-for-android).
