---
layout: post
category : lessons
tagline: "HTTPS GET requests in CFNetwork"
title: HTTPS CFNetwork GET Request
description: Performing low level HTTPS GET requests in Core Foundation
author: Kolin Stürt
tags : [iOS, Networking, CFNetwork, Core Foundation, HTTPS, SSL, TLS]
---
{% include JB/setup %}



## CFNetwork – HTTPS GET Requests in Core Foundation

There are few tutorials on the CFNetwork API and Apple's documentation is minimal, so here is a tutorial on Core Foundation's CFNetwork API.

**Disclaimer: It's always a good idea to start with the simplest solution to a problem and only move to a more complicated solution if need be. There may be times when the foundation framework is not available and we must use a lower level C class or a solution to a particular problem can not be addressed using one of the higher level frameworks. For some tasks such as network diagnostics, working with sockets and hosts, authenticating firewalls, working with ftp servers, or in order to debug SSL certificates, we must drop down to Core Foundation level. CFNetwork is much more configurable than it's foundation level network APIs and it's focus is on network protocols and providing a certain level of control over them; making it easier to operate than implementing BSD sockets. If you don't need this fine grained control, it's faster and less error prone to stick with something such as NSURLConnection or the AFNetworking library.**

For those of you that are looking towards using the CFNetwork library, this tutorial will start with a simple GET request. First lets create an URL. 


	CFURLRef theURL = CFURLCreateWithString(kCFAllocatorDefault, CFSTR("https://httpbin.org/get?test=helloWorld"), NULL); 


Httpbin is a good place to go for network test calls. This URL - https://httpbin.org/get will return to us  the GET data that we send it.

*Note:
Most CoreFoundation functions that create new objects start with the first parameter of the function pointing to the memory allocator. For now we will access the default memory allocator which is defined with the constant kCFAllocatorDefault*

*Note:
The second parameter of this function is for the URL string. CFSTR() is a nice macro to quickly create CFStringRef objects.*


Now lets create an HTTP request for the URL that we just set up. A request can also be called a “message”; simply as in sending a message over the network:


	CFHTTPMessageRef requestMessage = CFHTTPMessageCreateRequest(kCFAllocatorDefault, CFSTR("GET"), theURL, kCFHTTPVersion1_1);

For the parameters we have some options:

The second parameter is a string to the request method such as POST, PUT, or DELETE. Here we are using GET for the request type. Next we pass in our URL object that we created, and finally a string describing the most recent HTTP version 1.1. There is a constant for this,  kCFHTTPVersion1_1.


Because of the create-rule, the function that we used to create our URL will retain the returned object and since we are done using it, it is safe to release it now:

	CFRelease(theURL);


*NOTE: It is always a good habit to release or free any memory as soon as you are finished with it as opposed to at some later time such as at the end of the function. Memory can add up, and depending on the context of the application and state of the machine this could really matter!*

So far this process has been the same as a higher level framework for creating an URL and adding it to a request object. The difference here is at the CFNetwork layer, we deal directly with the underlying input and output streams on the network. So, in order to send this request, we must create a stream for the HTTP request we have just created.


	CFReadStreamRef readStream = CFReadStreamCreateForHTTPRequest(kCFAllocatorDefault, requestMessage);

Now that we are done using our request message we can safely release it here:

	CFRelease(requestMessage);


Now we have a stream ready to run! But before we go any further lets talk about streams. 

### Streams

Streams are very powerful. In fact, we can read or write bytes to a general network socket, to a file, or some part of memory. By default they are synchronous in that whatever thread you read or write from, that thread will block while the operation is in progress. In our case, just reading from a stream will block our main thread and the application will not be responsive. One way around this would be to spawn a thread. A more easy approach would be to use run loops. Underneath the hood we are still working with threads when using run loops. To understand more about run loops, go [here](https://developer.apple.com/library/ios/documentation/cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)


This tutorial has decided to run the network request asynchronously by scheduling it on a runloop that usually takes care of network requests. Our main thread and UI will not be blocked, and when the runloop receives activity such as returned data from the server, a callback will be called where we can handle the data.

Callbacks in CoreFoundation are very common. If you have used delegates in Foundation, you can think of this as a more global approach - a function is being called when some action takes place.


Lets schedule this stream with a run loop:


	CFReadStreamScheduleWithRunLoop(readStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);


We schedule the stream in the current run loop. The current run loop will thread the operation of the read in the background.

*NOTE:
Each run loop can run in a mode; different modes dictate a specific group of input sources and timers that will monitored, and observers that will be notified. There is already a set of common modes,  kCFRunLoopCommonModes, that we will use here.*

Setting a “client” for the read stream really means setting a “call back” function. To tell the read stream what function to call back we will use the CFReadStreamSetClient function:

	Boolean CFReadStreamSetClient(CFReadStreamRef stream, CFOptionFlags streamEvents, CFReadStreamClientCallBack clientCB, CFStreamClientContext *clientContext);


*NOTE: You will notice depending on which API you are using that there are different types of Boolean values. BOOL, bool, Boolean, boolean_t, CFBooleanRef, etc. If you want to know more, take a look [here](https://nshipster.com/bool/) (Boolean is a  historic type dating back to older Mac OS systems such as Mac OS 9. While C and C++ use ""bool"", Core Foundation often uses the “Boolean” type. CFBooleanRef is used to package a Boolean into a container object; it's constants kCFBooleanTrue and kCFBooleanFalse can be added to a CFDictionaryRef for example, just as @(YES) and @(NO) can be added to an NSDictionary.)*


### Events

The second parameter takes a bitfield of events for which we want to listen for. [Here](https://developer.apple.com/library/mac/documentation/corefoundation/Reference/CFStreamConstants/Reference/reference.html#//apple_ref/doc/uid/TP40003362-CH3-BFACGACJ) are the various types we can listen for. See the CFStream Event Type Constants - CFStreamEventType part of the page. What we often want is to see when bytes are available or when an error or end has occurred.


	CFOptionFlags flags = (kCFStreamEventHasBytesAvailable | kCFStreamEventErrorOccurred | kCFStreamEventEndEncountered);


The next parameter is the name of the function that will be called back when these events occur. The last parameter takes a CFStreamClientContext structure. This structure can be used to pass an additional object into the callback function, as well as have control over the memory management of that object. The structure looks like this:



	typedef struct {

    	CFIndex version;

    	void *info;

    	void *(*retain)(void *info);

    	void (*release)(void *info);

    	CFStringRef (*copyDescription)(void *info);

	} CFStreamClientContext;



So we can initialize this struct as follows:


	CFStreamClientContext context = {0, NULL, NULL, NULL, NULL};


Now that we have the event flags and context, lets set the callback:



	CFReadStreamSetClient(readStream, flags, GetRequestCallBack, &context); 


and now that everything is set up, lets start the request:



	CFReadStreamOpen(readStream);


Now lets write our callback function. The prototype for this callback function looks like this:


	void GetRequestCallBack(CFReadStreamRef readStream, CFStreamEventType type, void *clientCallBackInfo)


As you can see, in addition to having our stream passed back to us as the first parameter, we have the specific event (such as kCFStreamEventHasBytesAvailable) available to us and any context info we had previously passed in to the CFStreamClientContext structure.


In this tutorial we are simply going to read the stream into a data object. The CFReadStreamRead function happens to dumps bytes into a Uint8 * buffer. This is somewhat primitive, so we can at least append all those data bytes read into a nice data object – CFMutableDataRef. Lets create one now:


	CFMutableDataRef responseBytes = CFDataCreateMutable(kCFAllocatorDefault, 0);


The second parameter is the capacity limit of the data object. For now we use zero, which in CoreFoundation as well as in many other APIs, often translates to no limit.

Next we will start a loop and read data until there is no more data to be read. (In the next tutorial we will cover error handling and gathering data by the event type)


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



In this bit of code, we read from the stream using CFReadStreamRead. The second parameter is the buffer where the raw data is stored and the next parameter takes the size of that buffer. This function will return the number of bytes read. When the stream's end has arrived, this function returns 0. (and -1 if an error occurred). When the variable is zero, the loop exits. Inside the loop you will notice that while bytes have been read, we append those bytes into the CFMutableDataRef using CFDataAppendBytes. 

### Response 

This far we have created a request and then read the response into a data object, which is essentially complete in a bare bones type of way. However, you will notice high level APIs will return a response object, not just raw data. When a response is received from a server, the server also sends us response headers. We could package these headers and data all into one object - a response object.


	CFHTTPMessageRef response = (CFHTTPMessageRef)CFReadStreamCopyProperty(readStream, kCFStreamPropertyHTTPResponseHeader);



CFReadStreamCopyProperty is a function to get any properties from the object passed in, which in this case is the read stream. The second parameter is a constant asking the function to return a specific property, in this case the response headers.

Once we have created our response with the response headers, we can add that data we collected into this object using the CFHTTPMessageSetBody function. This function will retain the response bytes in the responseBytes data object so we can safely release the responseBytes variable afterwards.

	if (responseBytes)

    {

        if (response)

        {

           CFHTTPMessageSetBody(response, responseBytes);

        }

       CFRelease(responseBytes);

    }


Speaking of memory management, now let's be good citizens and do any leftover memory cleanup. We created the stream in the previous function so as with the create-rule we will need to release it. Since we are finished with the whole task at hand, we will also unschedule the stream and close it:



	CFReadStreamClose(readStream);
	CFReadStreamUnscheduleFromRunLoop(readStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);
	CFRelease(readStream);

### Testing

There is one more thing we should do before we end this tutorial. It would be nice to print out the response from the server as a string to the console to make sure everything is working. We can use the CFShow() function which will print out the a description of the object passed in to the console. Here is a function that will take the data object and create a string from it's bytes, and then print it to the console. Assumptions are made that this is a server response object for this example only:

	void LogResponseData(CFDataRef responseData)
	{
    	CFIndex dataLength = CFDataGetLength(responseData);
    	UInt8 *bytes = (UInt8 *)malloc(dataLength);
    	CFDataGetBytes(responseData, CFRangeMake(0, CFDataGetLength(responseData)), bytes);
    	CFStringRef responseString = CFStringCreateWithBytes(kCFAllocatorDefault, bytes, dataLength, kCFStringEncodingUTF8, TRUE);
    	CFShow(responseString);
    	CFRelease(responseString);
    	free(bytes);
	}

We can then add this to the end of our callback fuction:

	if (response)
    {
        CFDataRef responseBodyData = CFHTTPMessageCopyBody(response);
        CFRelease(response);
        
        LogResponseData(responseBodyData);
        CFRelease(responseBodyData);
    }

---

### Putting It All Together

Here is the full code snippet, as well as the entire XCode project for download [here](https://github.com/CollinStuart/CFNetworkGetRequest)

	void LogResponseData(CFDataRef responseData)
	{
	    CFIndex dataLength = CFDataGetLength(responseData);
	    UInt8 *bytes = (UInt8 *)malloc(dataLength);
	    CFDataGetBytes(responseData, CFRangeMake(0, CFDataGetLength(responseData)), bytes);
	    CFStringRef responseString = CFStringCreateWithBytes(kCFAllocatorDefault, bytes, dataLength, kCFStringEncodingUTF8, TRUE);
	    CFShow(responseString);
	    CFRelease(responseString);
	    free(bytes);
	}
	
	void GetRequestCallBack(CFReadStreamRef readStream, CFStreamEventType type, void *clientCallBackInfo)
	{
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
	    
	    //print response
	    if (response)
	    {
	        CFDataRef responseBodyData = CFHTTPMessageCopyBody(response);
	        CFRelease(response);
	        
	        LogResponseData(responseBodyData);
	        CFRelease(responseBodyData);
	    }
	}
	
	void GetRequest()
	{
	    CFURLRef theURL = CFURLCreateWithString(kCFAllocatorDefault, CFSTR("https://httpbin.org/get?test=helloWorld"), NULL);
	    CFHTTPMessageRef requestMessage = CFHTTPMessageCreateRequest(kCFAllocatorDefault, CFSTR("GET"), theURL, kCFHTTPVersion1_1);
	    CFRelease(theURL);
	    
	    CFReadStreamRef readStream = CFReadStreamCreateForHTTPRequest(kCFAllocatorDefault, requestMessage);
	    CFRelease(requestMessage);
	    
	    CFReadStreamScheduleWithRunLoop(readStream, CFRunLoopGetCurrent(), kCFRunLoopCommonModes);
	    
	    CFOptionFlags flags = (kCFStreamEventHasBytesAvailable | kCFStreamEventErrorOccurred | kCFStreamEventEndEncountered);
	    CFStreamClientContext context = {0, NULL, NULL, NULL, NULL};
	    CFReadStreamSetClient(readStream, flags, GetRequestCallBack, &context);
	    CFReadStreamOpen(readStream);
	}
