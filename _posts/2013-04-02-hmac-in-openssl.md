---
layout: post
category : lessons
tagline: "OpenSSL HMAC"
title: HMAC in OpenSSL
description: HMAC SHA512 with EVP in C++
author: Kolin Stürt
tags : [OpenSSL, C++, hash, SHA, message digest, hmac, C, how-to]
redirect_from:
  - /lessons/2013/04/02/hmac_in_openssl
---
{% include JB/setup %}

## HMAC in OpenSSL

In the [previous article](https://kolinsturt.github.io/lessons/2013/04/01/sha_using_openssl) you looked at creating a hash of data. Now you'll add a secret to the hash by using HMAC. HMAC authenticates data to make sure that it originated from the correct sender and that no one has altered the information. This works provided only you and the sender know the secret key.

You set up a key by calling `EVP_PKEY_new_mac_key`.

The `EVP_DigestSignInit` function takes a parameter for the type of hash function you'll use - SHA512, along with the key you create.

Then, `EVP_DigestSignUpdate` accepts the raw bytes and it's length of the data you wish to sign.

Consistent with all the other OpenSSL functions we've been using, `EVP_DigestSignFinal` finalizes that data. You'll need to cleanup the context with `EVP_MD_CTX_cleanup` to prevent memory leaks.
Make sure to also free the key that you created with `EVP_PKEY_free`.

    static vector<uint8_t> _rhfd(vector<uint8_t> const &plainData, string signingToken)
    {
        //sign using HMAC SHA512
        vector<uint8_t> signedData;
        if ( ( ! plainData.empty()) &&  ( ! signingToken.empty()))
        {
            signedData.resize(EVP_MAX_MD_SIZE);
            EVP_PKEY *key = EVP_PKEY_new_mac_key(EVP_PKEY_HMAC, NULL, (unsigned char *)signingToken.c_str(), (int)signingToken.length());
            if (key)
            {
                EVP_MD_CTX context;
                EVP_MD_CTX_init(&context);
                if (EVP_DigestSignInit(&context, NULL, EVP_sha512(), NULL, key))
                {
                    if (EVP_DigestSignUpdate(&context, plainData.data(), plainData.size()))
                    {
                        size_t lengthOfHash = 0;
                        if (EVP_DigestSignFinal(&context, signedData.data(), &lengthOfHash))
                        {
                            ;
                        }
                    }
                }
                EVP_MD_CTX_cleanup(&context);
                EVP_PKEY_free(key);
            }
        }

        return signedData;
    }
    
Newer versions recommend heap-based allocation:

    EVP_MD_CTX *context = NULL;
    context = EVP_MD_CTX_create();

That initializes the context. For stack-based you'll need to initialize it explicitly:

    EVP_MD_CTX context;
    EVP_MD_CTX_init(&context);
 
In the above example, `EVP_DigestSignInit` will cover initialization for you. For consistency, I'm leaving `EVP_MD_CTX_init` in the code as a reminder in case the implementation changes.

For good security, use at least a 256-bit key generated from a cryptographically secure random number generator. 

The HMAC CPU operation is fast, but because HMAC relies on a single shared key, you'll need to exchange the secret key securely. While there are ways to secure data in transit (iOS and Android example), it's not foolproof. If security is a priority over speed, the solution is to generate a key that doesn't need to leave the device in the first place. See the [next tutorial](https://kolinsturt.github.io/lessons/2013/04/03/ecdsa_in_openssl) to learn how to sign data using Public Key Cryptography.
