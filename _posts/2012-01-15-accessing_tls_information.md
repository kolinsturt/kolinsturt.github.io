---
layout: post
category : lessons
tagline: "CFNetwork Validating TLS"
description: Examining SSL/TLS certificate information in Core Foundation's CFNetwork. By Kolin Stürt
tags : [iOS, SSL, TLS Networking, CFNetwork, Core Foundation]
---
{% include JB/setup %}

## Accessing and Validating TLS information.

TLS (Transport Layer Security) and SSL (Secure Sockets Layer) are the main protocols used to provide encryption and authentication over network transactions. The TLS and SSL terms are often interchangeable and generally speak of the same thing. SSL 3.0 was actually the foundation of TLS 1.0. Some still refer to this as SSL 3.1. In the context of this tutorial, we will use TLS to refer to the most recent version of the cryptographic protocol. Normally, when an application is making a TLS connection, the operating system will verify the TLS certificate's chain of trust down to the known anchor certificate. If any of the certificates along the way are not valid, then the entire certificate is said to be invalid.

One of the reasons for working at the Core Foundation level is to be able to examine and verify the TLS certificate information of network requests. Often during development, we may use self signed certificates that would normally fail validation, create our own client certificates or connect directly to another computer via IP address. In these cases we could override the validation for development purposes, for example. In order to get started in examining the validation process, first we must include the security framework.

	#include <Security/Security.h>

### SecTrust

In Core Foundation, validation is executed by a configurable SecTrustRef object.
It is specifically created to assist in trust evaluation and contains various flags to control the type of validation performed as well as a policy, a SecPolicyRef object, that lets us change the hostname for TLS evaluation. The SecPolicyRef and TLS protocol use the X.509 standard, which depicts standard formats for certificates. X.509 also defines a standard for the certification path validation algorithm that validates the certificate path for a specific public key infrastructure. We will use this format to get information about our TLS connection.

The validation will happen in the callback function, the CFReadStreamClientCallBack for your CFNetwork stream. For information about setting up a basic CFNetwork request and stream, see the [GET](http://collinbstuart.github.io/lessons/2011/12/29/CFNetwork) and [POST](https://github.com/CollinBStuart/CFNetworkPostRequest) tutorials. Lets create a SecPolicy object now:

	SecPolicyRef policy = SecPolicyCreateBasicX509();

We can now use the read stream's CFReadStreamCopyProperty to get properties about our stream. We previously used this function to get the response headers in the [first tutorial](http://collinbstuart.github.io/lessons/2011/12/29/CFNetwork). This time we will use it to grab the TLS/SSL certificate using the constant kCFStreamPropertySSLPeerCertificates. This will return an array, a CFArrayRef of certificates. The wording of the function for the API uses SSL as opposed to TLS.

Once we get the certificate, a SecCertificateRef, we can create that SecTrustRef object with the certificate using the SecTrustCreateWithCertificates function.

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

We can get a lot of information about both the certificates and the policy by logging the object using the CFShow() function. Here we logged the sslArray and policy object.

### SecTrustEvaluate

Now that we have a SecTrustRef, we can evaluate it.
For code readability, lets create a separate function to parse the response.

	void ParseResponse(CFHTTPMessageRef response, SecTrustRef trust)
	
	
**NOTE: It's generally a good practice to have the length of your functions about one screen “page” for clarity. Each function should only do one task and be named appropriately. Then it is clear what each function actually does. If a function says “ComputeValue”, it's expected to do just that. If it also updates a database of user records, that should be in a separate function, otherwise you might want to call your function “ComputeValueAndUpdateUserRecords”. If the length of a function feels like it is getting too long, it's likely a clue that the function is doing more than one task and might benefit from being split up into multiple parts.**
	
Inside this function, we will call the SecTrustEvaluate function.

	OSStatus SecTrustEvaluate (SecTrustRef trust, SecTrustResultType *result);

This function verifies all the certificates in the certificate chain all the way up to the anchor certificate and also verifies the signature of the certificate. The second parameter to the function is of a SecTrustResultType. Somewhat misleading in its name, the most common success result is kSecTrustResultUnspecified - which means that all certificates passed validation, but did not reach any certificate that was explicitly chosen by the user to be trusted. If verification fails, the kSecTrustResultRecoverableTrustFailure, kSecTrustResultFatalTrustFailure, or kSecTrustResultOtherError will be stored into "result". kSecTrustResultRecoverableTrustFailure is often called when one of the certificates in the chain is expired or the anchor is not in the set of trusted anchors.  kSecTrustResultFatalTrustFailure usually occurs when there is a problem interpreting the data or extensions of the certificate. kSecTrustResultOtherError happens when some other operating system level error has occurred or a certificate was revoked. 

Additionally, the function will return an OSStatus code. Some of the codes refer to invalid keychains or settings, for example. See the ""Result Codes" section of the [Certificate, Key, and Trust Services Reference](https://developer.apple.com/library/mac/documentation/security/Reference/certifkeytrustservices/Reference/reference.html#//apple_ref/doc/uid/TP30000157-CH4g-CJBEABHG) for more information about possible error codes.

The full code looks like this:

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

To create a simple TLS request, all we need to do is append "s" to the http section of the URL. As long as the host supports TLS and has valid certificates, we will get certificate information back. Appending "s" to "https" to create a secure connection is the same for both CFNetwork and higher level APIs such as NSURLConnection and AFNetworking. Here is an example request. The very first tutorial explained setting up a basic URL request. So for more information about how this works, see [here](http://collinbstuart.github.io/lessons/2011/12/29/CFNetwork).

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

As a final note to this tutorial. TLS evaluation generally will fail for self signed certificates or an intermediate certificate is missing and by overriding validation we can ignore the failure. Make sure you do not ship production code to override evaluation in this way. Often evaluation fails because of a malicious intent and therefor its best to make sure not to override evaluation in production code.

For more detailed information and documentation, check out the [Certificate, Key and Trust Services](https://developer.apple.com/library/mac/documentation/security/Reference/certifkeytrustservices/Reference/reference.html)

The full source code in an XCode project for this tutorial can be found [here](https://github.com/CollinBStuart/TLSValidation).
