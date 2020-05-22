---
layout: post
category : lessons
tagline: "HTTPS CFNetwork POST Request"
title: HTTPS requests in CFNetwork
description: Performing low level HTTPS POST requests in Core Foundation
author: Kolin Stürt
tags : [iOS, Networking, CFNetwork, Core Foundation, HTTPS, SSL, TLS]
---
{% include JB/setup %}

## HTTPS POST Request With Headers

In this tutorial we are going to create an HTTPS POST request as well as add some request headers using the CFNetwork API. This article assumes some prior understanding of Core Foundation which was discussed in the first tutorial [Simple GET Requests in Core Foundation](https://collinstuart.github.io/lessons/2011/12/29/CFNetwork/). 

### Adding POST data
To get started, first lets add a post value as a string, then turn it into an external representation - a CFData object that is packaged to be sent out over the network:

    // Create the POST request payload.
    CFStringRef payloadString = CFStringCreateWithFormat(kCFAllocatorDefault, NULL, CFSTR("{\"the-data-key\" : \"my-data-value\"}"));
    CFDataRef payloadData = CFStringCreateExternalRepresentation(kCFAllocatorDefault, payloadString, kCFStringEncodingUTF8, 0);
    CFRelease(payloadString);

Next we create a network request, just as we did in the [first tutorial](https://collinstuart.github.io/lessons/2011/12/29/CFNetwork/).

    //create request
    CFURLRef theURL = CFURLCreateWithString(kCFAllocatorDefault, CFSTR("https://httpbin.org/post"), NULL); //https://httpbin.org/post returns post data
    CFHTTPMessageRef request = CFHTTPMessageCreateRequest(kCFAllocatorDefault, CFSTR("POST"), theURL, kCFHTTPVersion1_1);
    CFHTTPMessageSetBody(request, payloadData);

### Adding Request headers

Now lets add some headers. While POST values are used to send data enclosed inside the request's body to a server, header fields are used to control operating parameters of the HTTP transaction itself; they are processed by the HTTP server before the server relays the data to be processed. To add headers to a request, we use the CFHTTPMessageSetHeaderFieldValue function. The second parameter to this function takes the header field name while the third parameter contains the value for the header field name. Here we will set three header values: the HOST, Content-Length, and Content-Type. HOST usually contains the name of the server as well as the port number, while Content-Length is the length of the body of the request, and Content-Type specifies the MIME type of the request body. See the [List of HTTP header fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields#Field_values) for more info.

    //add some headers
    CFStringRef hostString = CFURLCopyHostName(theURL);
    CFHTTPMessageSetHeaderFieldValue(request, CFSTR("HOST"), hostString);
    CFRelease(hostString);
    CFRelease(theURL);
    
    if (payloadData)
    {
        CFStringRef lengthString = CFStringCreateWithFormat(kCFAllocatorDefault, NULL, CFSTR("%ld"), CFDataGetLength(payloadData));
        CFHTTPMessageSetHeaderFieldValue(request, CFSTR("Content-Length"), lengthString);
        CFRelease(lengthString);
    }
    
    
    CFHTTPMessageSetHeaderFieldValue(request, CFSTR("Content-Type"), CFSTR("charset=utf-8"));



Next we will setup a network stream for our request, the same way we did in the [first tutorial](https://collinstuart.github.io/lessons/2011/12/29/CFNetwork/). The first tutorial goes over line by line exactly what each function does.

    //create read stream for response
    CFReadStreamRef requestStream = CFReadStreamCreateForHTTPRequest(kCFAllocatorDefault, request);
    CFRelease(request);

    //set up on separate runloop (with own thread) to avoid blocking the UI
    CFReadStreamScheduleWithRunLoop(requestStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);
    CFOptionFlags optionFlags = (kCFStreamEventHasBytesAvailable | kCFStreamEventErrorOccurred | kCFStreamEventEndEncountered);

### Providing a Data Context for the Callback

Now lets look at creating our callback function that will be invoked when data has arrived back from our server from the POST request. Lets expand on what we have learned in the first tutorial. Last time when we set up our callback, there was an option in the CFStreamClientContext structure to include some contextual information that can be passed back to the callback function and we had ignored it. This time lets take the opportunity to add some contextual information. Just as an example, lets add our POST data as a context object.


    CFStreamClientContext clientContext = {0, (void *)payloadData, RetainSocketStreamHandle, ReleaseSocketStreamHandle, NULL};
    CFReadStreamSetClient(requestStream, optionFlags, ReadStreamCallBack, &clientContext); 

Here is the definition of the CFStreamClientContext structure

	struct CFStreamClientContext {
	   CFIndex version;
	   void *info;
	   void *(*retain)(void *info);
	   void (*release)(void *info);
	   CFStringRef (*copyDescription)(void *info);
	} CFStreamClientContext;

### Retain and Release callback functions

The third and forth items in the struct defines specific memory management behaviour for the data context and the last item is used for setting a callback function that returns a CFString object describing the characteristics of the data passed in. We can leave the retain and release items as NULL and nothing would be done to the memory management of the object we pass in. If we set the copyDescription pointer to NULL the default description function is used. We can pass in default CFRetain, CFRelease, and the default description function CFCopyDescription right into the structure to explicitly declare to use the default functions:

    CFStreamClientContext clientContext = {0, (void *)payloadData, (void *(*)(void *info))CFRetain, (void (*)(void *info))CFRelease, (CFStringRef (*)(void *info))CFCopyDescription};

We can also pass a pointer to our own custom function callback that would retain and release the data in the info variable. You can think of this as overriding the default functions in order to provide more functionality upon retaining or releasing, for example. As an exercise, we can take this opportunity to define simple retain and release callback functions. For this example, all the code does is retain and release the info object in the same way that the default functions would behave.

	void *RetainSocketStreamHandle(void *info)
	{
    	     CFRetain(info);
    	     return info;
	}

	void ReleaseSocketStreamHandle(void *info)
	{
    	    if (info)
    	    {
        	CFRelease(info);
    	    }
	}

### Putting It All Together

The last thing to do is to open the stream. Our full post request code should look like this:

	void PostRequest()
	{
	    // Create the POST request payload.
	    CFStringRef payloadString = CFStringCreateWithFormat(kCFAllocatorDefault, NULL, CFSTR("{\"test-data-key\" : \"test-data-value\"}"));
	    CFDataRef payloadData = CFStringCreateExternalRepresentation(kCFAllocatorDefault, payloadString, kCFStringEncodingUTF8, 0);
	    CFRelease(payloadString);
	    
	    //create request
	    CFURLRef theURL = CFURLCreateWithString(kCFAllocatorDefault, CFSTR("https://httpbin.org/post"), NULL); //https://httpbin.org/post returns post data
	    CFHTTPMessageRef request = CFHTTPMessageCreateRequest(kCFAllocatorDefault, CFSTR("POST"), theURL, kCFHTTPVersion1_1);
	    CFHTTPMessageSetBody(request, payloadData);
	    
	    //add some headers
	    CFStringRef hostString = CFURLCopyHostName(theURL);
	    CFHTTPMessageSetHeaderFieldValue(request, CFSTR("HOST"), hostString);
	    CFRelease(hostString);
	    CFRelease(theURL);
	    
	    if (payloadData)
	    {
	        CFStringRef lengthString = CFStringCreateWithFormat(kCFAllocatorDefault, NULL, CFSTR("%ld"), CFDataGetLength(payloadData));
	        CFHTTPMessageSetHeaderFieldValue(request, CFSTR("Content-Length"), lengthString);
	        CFRelease(lengthString);
	    }
	    
	    
	    CFHTTPMessageSetHeaderFieldValue(request, CFSTR("Content-Type"), CFSTR("charset=utf-8"));
	    
	    //create read stream for response
	    CFReadStreamRef requestStream = CFReadStreamCreateForHTTPRequest(kCFAllocatorDefault, request);
	    CFRelease(request);
	    
	    //set up on separate runloop (with own thread) to avoid blocking the UI
	    CFReadStreamScheduleWithRunLoop(requestStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);
	    CFOptionFlags optionFlags = (kCFStreamEventHasBytesAvailable | kCFStreamEventErrorOccurred | kCFStreamEventEndEncountered);
	    CFStreamClientContext clientContext = {0, (void *)payloadData, RetainSocketStreamHandle, ReleaseSocketStreamHandle, NULL};
	    CFReadStreamSetClient(requestStream, optionFlags, ReadStreamCallBack, &clientContext);
	    
	    //start request
	    CFReadStreamOpen(requestStream);
	    
	    if (payloadData)
    	    {
        	CFRelease(payloadData);
    	    }
	}

Here is our callback function that will be invoked when the read stream receives new data, the end of the data, or an error. This function will log to the console the data we passed in as our context, and then log the response received from the server which should also include our POST data we sent. We used the callback function of the [previous tutorial](https://collinstuart.github.io/lessons/2011/12/29/CFNetwork/) as a template which you can refer to for details of how each line works.

	void LogData(CFDataRef responseData)
	{
	    CFIndex dataLength = CFDataGetLength(responseData);
	    UInt8 *bytes = (UInt8 *)malloc(dataLength);
	    CFDataGetBytes(responseData, CFRangeMake(0, CFDataGetLength(responseData)), bytes);
	    CFStringRef responseString = CFStringCreateWithBytes(kCFAllocatorDefault, bytes, dataLength, kCFStringEncodingUTF8, TRUE);
	    CFShow(responseString);
	    CFRelease(responseString);
	    free(bytes);
	}
	
	static void ReadStreamCallBack(CFReadStreamRef readStream, CFStreamEventType type, void *clientCallBackInfo)
	{
	    CFDataRef passedInData = (CFDataRef)(clientCallBackInfo);
	    CFShow(CFSTR("Passed In Data:"));
	    LogData(passedInData);
	    
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
	    
	    //once all data is appended, package it all together - create a response from the response headers, and add the data received.
	    //note: just having the data received is not enough, you need to finish the response by retrieving the response headers here...
	    CFHTTPMessageRef response = (CFHTTPMessageRef)CFReadStreamCopyProperty(readStream, kCFStreamPropertyHTTPResponseHeader);
	    
	    if (responseBytes)
	    {
	        if (response)
	        {
	            CFHTTPMessageSetBody(response, responseBytes);
	        }
	        CFRelease(responseBytes);
	    }
	    
	    
	    //close and cleanup
	    CFReadStreamClose(readStream);
	    CFReadStreamUnscheduleFromRunLoop(readStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);
	    CFRelease(readStream);
	    
	    //just keep the response body and release requests
	    CFDataRef responseBodyData = CFHTTPMessageCopyBody(response);
	    if (response)
	    {
	        CFRelease(response);
	    }
	    
	    //get the response as a string
	    if (responseBodyData)
	    {
	        CFShow(CFSTR("\nResponse Data:"));
	        LogData(responseBodyData);
	        CFRelease(responseBodyData);
	    }
	}
	
And that's it for creating POST requests with headers and a context. Don't forget that so far we have not covered error handling. The full XCode project can be downloaded [here](https://github.com/CollinBStuart/CFNetworkPostRequest).
