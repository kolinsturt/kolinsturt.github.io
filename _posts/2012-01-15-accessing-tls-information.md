---
layout: post
category : lessons
tagline: "CFNetwork TLS"
title:  Customizing TLS Server Trust Evaluation
description: Examining and customizing TLS certificate information in CFNetwork
author: Kolin Stürt
tags : [iOS, SSL, TLS Networking, CFNetwork, Core Foundation, SecTrustEvaluate]
---
{% include JB/setup %}

## Customizing TLS Server Trust Evaluation

TLS (Transport Layer Security) and SSL (Secure Sockets Layer) are the main protocols used to provide encryption and authentication over network transactions. The TLS and SSL terms are often interchangeable and generally speak of the same thing. SSL 3.0 was actually the foundation of TLS 1.0. Some still refer to this as SSL 3.1. In this article, you'll use TLS to refer to the most recent version of the cryptographic protocol. When an application is making a TLS connection, the operating system will verify the TLS certificate’s chain of trust down to the known anchor certificate. If any of the certificates along the way are not valid, then the entire certificate is invalid.

One of the reasons for working at the Core Foundation level is to to examine and verify the TLS certificate information of network requests. During development, you may use self signed certificates that would fail validation, create your own client certificates or connect to another computer via an IP address. In these cases you could override the validation for development purposes, for example. You should never override this system for production environments.

On the other hand, the Core Foundation callbacks occur when the OS has already established a TLS connection. Therefor network solutions such as Certificate Pinning are not avaiable at this abstraction. If you're interested in TLS pinning, check out the [Securing Communications on iOS](http://code.tutsplus.com/articles/securing-communications-on-ios--cms-28529) and [Securing Network Data Tutorial for Android](https://www.raywenderlich.com/10056112-securing-network-data-tutorial-for-android).

To get started in examining the validation process, first include the security framework:

	#include <Security/Security.h>

### Working With SecTrust

Core Foundation performs validation via a configurable `SecTrustRef` object. It assists in trust evaluation and contains flags to control the type of validation performed. A `SecPolicyRef` object lets you change the hostname for TLS evaluation. The `SecPolicyRef` and TLS protocol use the X.509 standard, which depicts standard formats for certificates. X.509 also defines a standard for the certification path validation algorithm that validates the certificate path for a specific public key infrastructure. You'll this format to get information about the TLS connection.

The validation will happen in the callback function, the `CFReadStreamClientCallBack` for your CFNetwork stream. For information about setting up a basic CFNetwork request, see the [GET](http://collinbstuart.github.io/lessons/2011/12/29/CFNetwork) and [POST](https://github.com/CollinBStuart/CFNetworkPostRequest) tutorials. Create a `SecPolicy` object now:

	SecPolicyRef policy = SecPolicyCreateBasicX509();

You can now use `CFReadStreamCopyProperty` to get properties about the stream. You used this function to get the response headers in [another tutorial](http://collinbstuart.github.io/lessons/2011/12/29/CFNetwork). This time you'll use it to grab the TLS/SSL certificate using the `kCFStreamPropertySSLPeerCertificates` constant. It will return a `CFArrayRef` of certificates. The terms for the functions use SSL as opposed to TLS.

Once you get the `SecCertificateRef`, you can create the `SecTrustRef` object with the certificate using the `SecTrustCreateWithCertificates` function:

    SecTrustRef trust = NULL;
    SecPolicyRef policy = SecPolicyCreateBasicX509();
    CFArrayRef sslArray = CFReadStreamCopyProperty(readStream, kCFStreamPropertySSLPeerCertificates);
    
    CFShow(CFSTR("SSL Array:"));
	CFShow(sslArray);
	    
    if (sslArray)
    {
        if (CFArrayGetCount(sslArray)) 
        {
            SecTrustCreateWithCertificates(sslArray, policy, &trust);
            CFShow(CFSTR("\nPolicy is:"));
	        CFShow(policy);
        }
        CFRelease(sslArray);
    }
    CFRelease(policy);

You can get a lot of information about the certificates and the policy using the `CFShow()` function. Here you logged the `sslArray` and `policy` object.

### Calling SecTrustEvaluate

Now that you have a `SecTrustRef`, you can evaluate it. For readability, create a separate function to parse the response:

	void ParseResponse(CFHTTPMessageRef response, SecTrustRef trust)
	
Inside that function, call `SecTrustEvaluate`:

	OSStatus SecTrustEvaluate (SecTrustRef trust, SecTrustResultType *result);

This function verifies all the certificates in the certificate chain all the way up to the anchor certificate. It verifies the signature of the certificate. The second parameter to the function is of a `SecTrustResultType`. Misleading in its name, you should expect the "success" result `kSecTrustResultUnspecified`. It means that all certificates passed validation. "Unspecified" means it didn't reach a certificate that you explicitly chose for it to trust. If verification fails, it stores `kSecTrustResultRecoverableTrustFailure`, `kSecTrustResultFatalTrustFailure` or `kSecTrustResultOtherError` into `result`.

- `kSecTrustResultRecoverableTrustFailure` is due to an expired certificate or the anchor is not in the set of trusted anchors.
- `kSecTrustResultFatalTrustFailure` occurs when there is a problem interpreting the data or extensions of the certificate.
- `kSecTrustResultOtherError` happens for a revoked certificate or an operating system level error.
- The function also returns an `OSStatus` code. Some of the codes refer to invalid keychains or settings.
- See the **Result Codes** section of the [Certificate, Key, and Trust Services Reference](https://developer.apple.com/library/mac/documentation/security/Reference/certifkeytrustservices/Reference/reference.html#//apple_ref/doc/uid/TP30000157-CH4g-CJBEABHG) for possible error codes.

Here's the full code:

	void ParseResponse(CFHTTPMessageRef response, SecTrustRef trust)
	{
	    SecTrustResultType trustResult;
	    
	    if ((errSecSuccess == SecTrustEvaluate(trust, &trustResult)) && (trustResult == kSecTrustResultUnspecified))
	    {
	        CFShow(CFSTR("Evaluate Succeeded!"));
	    }
	    else
	    {
	        CFShow(CFSTR("Evaluate Failed."));
	    }
	    
	    if (trust)
	    {
	        CFRelease(trust);
	    }
	    if (response)
	    {
	        CFRelease(response);
	    }
	}

	static void ReadStreamCallBack(CFReadStreamRef readStream, CFStreamEventType type, void *clientCallBackInfo)
	{
	    
	    //append data as we receive it
	    CFMutableDataRef responseBytes = CFDataCreateMutable(kCFAllocatorDefault, 0);
	    CFIndex numberOfBytesRead = 0;
	    do
	    {
	        UInt8 buf[1024];
	        numberOfBytesRead = CFReadStreamRead(readStream, buf, sizeof(buf));
	        if (numberOfBytesRead > 0)
	        {
	            CFDataAppendBytes(responseBytes, buf, numberOfBytesRead);
	        }
	    } while (numberOfBytesRead > 0);
	    
	    //package appended data into a response object
	    CFHTTPMessageRef response = (CFHTTPMessageRef)CFReadStreamCopyProperty(readStream, kCFStreamPropertyHTTPResponseHeader);
	    
	    if (responseBytes)
	    {
	        if (response)
	        {
	            CFHTTPMessageSetBody(response, responseBytes);
	        }
	        CFRelease(responseBytes);
	    }
	    
	    SecTrustRef trust = NULL;
	    SecPolicyRef policy = SecPolicyCreateBasicX509();
	    CFArrayRef sslArray = CFReadStreamCopyProperty(readStream, kCFStreamPropertySSLPeerCertificates);
	    
	    CFShow(CFSTR("SSL Array:"));
	    CFShow(sslArray);
	    
	    if (sslArray)
	    {
	        if (CFArrayGetCount(sslArray))
	        {
	            SecTrustCreateWithCertificates(sslArray, policy, &trust);
	            CFShow(CFSTR("\nPolicy is:"));
	            CFShow(policy);
	        }
	        CFRelease(sslArray);
	    }
	    CFRelease(policy);
		
	    //close and cleanup
	    CFReadStreamClose(readStream);
	    CFReadStreamUnscheduleFromRunLoop(readStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);
	    CFRelease(readStream);
	    
	    ParseResponse(response, trust);
	}

### Putting It All Together

To create a simple TLS request, all you need to do is append **s** to the **http** section of the URL. As long as the host supports TLS and has valid certificates, you'll get certificate information back. Appending **s** to **http** to create a secure connection is the same for both CFNetwork and higher level APIs such as NSURLConnection. Here's an example request. The very first tutorial explained setting up a basic URL request. So for more information about how this works, see [here](https://kolinsturt.github.io/lessons/2011/12/29/CFNetwork).

	void TLSRequest()
	{
	    CFURLRef theURL = CFURLCreateWithString(kCFAllocatorDefault, CFSTR("https://httpbin.org"), NULL);
	    CFHTTPMessageRef requestMessage = CFHTTPMessageCreateRequest(kCFAllocatorDefault, CFSTR("GET"), theURL, kCFHTTPVersion1_1);
	    CFRelease(theURL);
	    
	    CFReadStreamRef readStream = CFReadStreamCreateForHTTPRequest(kCFAllocatorDefault, requestMessage);
	    CFRelease(requestMessage);
	    
	    CFReadStreamScheduleWithRunLoop(readStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);
	    
	    CFOptionFlags flags = (kCFStreamEventHasBytesAvailable | kCFStreamEventErrorOccurred | kCFStreamEventEndEncountered);
	    CFStreamClientContext context = {0, NULL, NULL, NULL, NULL};
	    CFReadStreamSetClient(readStream, flags, ReadStreamCallBack, &context);
	    CFReadStreamOpen(readStream);
	}

TLS evaluation will fail for self signed certificates. It will also fail if an intermediate certificate is missing. By overriding validation you can ignore the failure. Make sure you do not ship production code to override evaluation in this way. Often evaluation fails because of a malicious intent. It's never recommended ti override evaluation in production code.

For more detailed information and documentation, check out the [Certificate, Key and Trust Services](https://developer.apple.com/library/mac/documentation/security/Reference/certifkeytrustservices/Reference/reference.html)

The full source code in an XCode project for this tutorial is [here](https://github.com/CollinBStuart/TLSValidation).

Again, the callbacks for this API happen when the system has already made a TLS connection. Therefor pre-validation solutions such as SSL Pinning are not avaiable at this abstraction. If you're interested in certificate pinning, check out the [Securing Communications on iOS](http://code.tutsplus.com/articles/securing-communications-on-ios--cms-28529) and [Securing Network Data Tutorial for Android](https://www.raywenderlich.com/10056112-securing-network-data-tutorial-for-android).
