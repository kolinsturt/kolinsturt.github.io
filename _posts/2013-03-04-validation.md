---
layout: post
category : lessons
tagline: "Input validation"
title: App hardening with input validation
description: Input validation for app hardening
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
---
{% include JB/setup %}

## Validation

User input validation is the strongest countermeasure to some of the most common security vulnerabilities. Many of the vulnerabilities such as buffer overflows can be prevented by checking all places in an application where a user can input data. Lets go through some of the common Foundation vulnerabilities.

### NSFileHandle

NSFileHandle is vulnerable to race conditions. It's method fileHandleForWritingAtPath returns an object when there is already a file at the path. Creating and opening the file happens in two separate steps so that a malicious user can change the created file with other data before the file is read. For example, the file can be changed to link to another file so that when the application goes to write to the file, it is instead overwriting an unknown file. It is better to create a temporary file with mktemp() and use CFFileDescriptor's CFFileDescriptorCreate, or NSFileHandle's initWithfileDescriptor.

	fd = mkstemp(tmpfile); // check for -1 error
	NSFileHandle *myhandle = [[NSFileHandle alloc] initWithFileDescriptor:fd];
	
### Null Pointers	
	
Sanitize and validate files coming from any location outside the app sandbox. In addition to checking sizes, it is important, especially when receiving data from a server, to check that it is the correct type. Make sure to always check for NULL. Most CoreFoundation functions will crash if you try operating on NULL. For example, CFRelease will crash if it is passed NULL so make sure arrays, dictionaries and other created objects have successfully been created before releasing.

	if (array)
	{
		CFRelease(array);
		array = NULL;
	}
	
If you assign your ivars to NULL or nil after use, then you can safely check if an object exists as in the above example. If you do not set ivars to NULL, they may be reused after releasing and may point to some unknown object. The only side effect of settings all ivars to NULL is that it may hide if there is a problem with the logic of the code. A crash in development mode is good in that it draws attention to an underlying problem.

### Dynamic Type Checking

Each Core Foundation object has a type ID. While there isn't a specific list of types that should be checked against, to check for a specific type of object, use the CFGetTypeID() on the object in question and compare it with the object you want to check for. For example, if you want to make sure the object is a string, use CFStringGetTypeID. Each Core foundation object has a CFxGetTypeID function. To check for CFNullRef's do this

	if ( CFGetTypeID(data) != CFNullGetTypeID() )

In Foundation, you can do this

	if ([someObject isKindOfClass: [NSArray class]]) 

Additionally, you can see if an object conforms to a protocol before trying to send messages to it

	if ([someObject conformsToProtocol: @protocol(MyProtocol)]) 

And before you send a delegate a method, make sure the app will not crash on "unrecognized selector"

	if ([someObject respondsToSelector:@selector(someMethod)])

### Strings

Beyond validating and sanitizing data from a server, make sure you choose which information the user can see. For example, it is a bad idea to display an alert when encountering a server error and to just pass the error message through from the server. Error messages could disclose securely related information. For example, it may guide a trial and error attack against an area of the app that the user is not authorized to access.

Make sure you encode your strings before sending them to a server and visa versa. NSString's stringByAddingPercentEscapesUsingEncoding does not encode some characters such as ampersand and slashes. Therefore it might be best to write your own method using Core Foundation's percent escape functions where you have more fine grained control over what is encoded and decoded.

	+ (NSString *)stringByEscapingString:(NSString*)string //as per rfc3986 - note the other escaping string methods escape different characters according to different standards.
	{
	    //if not valid string
	    if ([self _isEmptyString:string])
	    {
	        //return
	        return @"";
	    }
	    
	    //add percent escapes. Use Core Foundation because Foundation has a bug that does not encode ampersand properly
	    CFStringRef encodedStringRef  = CFURLCreateStringByAddingPercentEscapes(NULL, 
	                                                                            (CFStringRef)string, NULL, 
	                                                                            CFSTR(":/?#[]@!$ &'()*+,;=\"<>%{}|\\^`"),
	                                                                            kCFStringEncodingUTF8);
	    
	    //create an autoreleased foundation object
	    NSString *encodedString = [NSString stringWithString:(NSString *)encodedStringRef];
	    
	    //cleanup
	    CFRelease(encodedStringRef);
	    
	    //return escaped string
	    return encodedString;
	}
	
	+ (NSString *)stringByDecodingString:(NSString *)string
	{
	    if ([self _isEmptyString:string])
	    {
	        return @"";
	    }
	    CFStringRef unescapedString = CFURLCreateStringByReplacingPercentEscapesUsingEncoding (NULL, (CFStringRef)string, CFSTR(""), kCFStringEncodingUTF8);
	    NSString *decodedString = [NSString stringWithString:(NSString *)unescapedString];
	    CFRelease(unescapedString);
	    return decodedString;
	}
	
Here we added error handling for empty strings. In order to truly test for empty strings, you must take into consideration that there might be whitespace and remove it
	
	+(BOOL)_isEmptyString:(NSString *)string
	{
	    //if we do not have a string
	    if ([[string stringByTrimmingCharactersInSet:[NSCharacterSet whitespaceCharacterSet]] length] == 0)
	    {
	        //string is empty
	        return YES;
	    }
	    else 
	    {
	        //string is not empty
	        return NO;
	    }
	}
	
Any other user input should be validated so that the input is of the expected type and range. For example, if a user is to enter an email address, we can check for a valid address

	+ (BOOL) validateEmailFromString:(NSString *)emailString useStrictValidation:(BOOL)isStrict
	{
	    NSString *filterString = nil;
	    if (isStrict)
	    {
	        filterString = @"[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,4}";
	    }
	    else
	    {
	        filterString = @".+@.+\\.[A-Za-z]{2}[A-Za-z]*";
	    }
	    
	    NSPredicate *emailPredicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@", filterString];
	    return [emailPredicate evaluateWithObject:emailString];
	}
	
We can also limit the number of characters typed in a textfield by setting the textfield's delegate and then implementing it's shouldChangeCharactersInRange delegate method

	-(BOOL) textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string
	{
	    NSUInteger newLength = [[textField text] length] + [string length] - range.length;
	    if (newLength > kMaxSearchLength)
	    {
	        return NO;
	    }
	
	    return YES;
	}
	
For a textview, the delegate method to implement is this

	- (BOOL)textView:(UITextView *)textView shouldChangeTextInRange:(NSRange)range replacementText:(NSString *)text;
	
### Images

If a user is uploading an image to the server, we can check that it is a valid image. For example, for a JPEG file, the first two bytes and last two bytes are always FF D8 and FF D9.
	
	+ (BOOL)validateImageData:(NSData *)data
	{
	    if (!data || data.length < 2)
	        return NO;
	    
	    NSInteger totalBytes = data.length;
	    const char *bytes = (const char*)[data bytes];
	    
	    BOOL isValid = (bytes[0] == (char)0xff &&
	                    bytes[1] == (char)0xd8 &&
	                    bytes[totalBytes-2] == (char)0xff &&
	                    bytes[totalBytes-1] == (char)0xd9);
	    
	    return isValid;
	    
	    if (!data || data.length < 12)
	    {
	        return NO;
	    }
	}
	
### Command Injection Attacks
	
Another point of entry that should be validated is any registered URL scheme that can send commands to the application. Make sure you don't accept commands that are complete URL or pathnames, or that directly get passed into your code that could cause buffer overflows or format string attacks.

Be extremely careful when any user input is passed into commands that will be executed by an SQL server or server that will run code. Make sure to remove any characters of the specific language that the server will be processing so that the input is not susceptible to any command injection attacks.

Another often overlooked area that should be validated is from unarchiving object graph serializations, or saved data objects (XIB and NIB files included). If the archived data is not stored or transferred securely, either data or even method implementations can be changed. Validation should be done in initWithCoder and only save or package classes that conform to NSSecureCoding. 

One more overlooked area is network callback functions and signals. If you are setting up a runloop so that an event will be triggered when data is received; such as a socket connection waiting for server data, you can never be sure that the server process hasn't been compromised. Therefor, do not assume in your program flow when you think the callback will be fired. Be prepared for it to happen anytime and also be very careful validating the input received. Make sure the read buffer will not overflow, or block the main thread resulting in a denial of service attack that was successfully launched from the server. POSIX Networking and CFNetworking are out of scope of this post, so see [this guide](http://collinbstuart.github.io/lessons/2013/01/01/CFSocket) for more details.
