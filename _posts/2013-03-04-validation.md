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

## Input Validation

In this article, you'll learn about secure coding in Objective-C. User input validation is the strongest countermeasure to some of the most common security vulnerabilities. You can prevent many of the vulnerabilities such as buffer overflows by checking all places in an application where a user can input data. Input text fields should have a max character limit and only accept the type of input expected.

### NSFileHandle

`NSFileHandle` is vulnerable to race conditions. It’s method `fileHandleForWritingAtPath` returns an object when there is already a file at the path. Creating and opening the file happens in two separate steps. A malicious user can change the created file before the system reads it. For example, you can change the file to link to another file. When the application goes to write to the file, it is instead overwriting a different one. It's safer to create a temporary file with `mktemp()`, use `CFFileDescriptor`’s `CFFileDescriptorCreate` or NSFileHandle’s `initWithfileDescriptor`:

	fd = mkstemp(tmpfile); // check for -1 error
	NSFileHandle *myhandle = [[NSFileHandle alloc] initWithFileDescriptor:fd];
	
### Null Pointers

Sanitize and validate files coming from any location outside the app sandbox. In addition to checking sizes, it is important, especially when receiving data from a server, to check that it is the correct type. Make sure to always check for `NULL`. Most Core Foundation functions will crash if you try operating on `NULL`. For example, `CFRelease` will crash if you pass in `NULL` so make sure objects exist before working with them:

	if (array)
	{
		CFRelease(array);
		array = NULL;
	}
	
If you assign your ivars to `NULL` or `nil` after use, then you can check if an object exists as in the above example. If you do not set ivars to `NULL`, the app may reuse them after release and may point to an unknown object. The only side effect of settings all ivars to `NULL` is that it may hide if there is a problem with the logic of the code. A crash in development mode is good in that it draws attention to an underlying problem.

### Dynamic Type Checking

Checking in Foundation is as easy as this:

	if ([someObject isKindOfClass: [NSArray class]]) 

You can see if an object conforms to a protocol before trying to send messages to it:

	if ([someObject conformsToProtocol: @protocol(MyProtocol)]) 
	
You can do the same with dynamic objects. For example, before you send a delegate a method, make sure the app will not crash on **unrecognized selector**:

	if ([someObject respondsToSelector:@selector(someMethod)])
	
	
For Core Foundation, each object has a type ID. There isn’t a specific list of types you can check against. To check for a specific type of object, use the `CFGetTypeID()` on the object in question and compare it with the object you want to check for. For example, if you want to make sure the object is a string, use `CFStringGetTypeID`. Each Core foundation object has a `CFxGetTypeID` function. To check for `CFNullRef` do this:

	if ( CFGetTypeID(data) != CFNullGetTypeID() )

### Strings

Beyond validating and sanitizing data from a server, make sure you choose which information the user can see. For example, it's a bad idea to display an alert when encountering a server error that passes the error message directly. Error messages could disclose security related information. A good example is when a login alert tells you if the username verses a password is incorrect. An attacker can iterate through usernames to find out if it exists on the system. A generalized message such as **your credentials are incorrect** solves this problem.

Make sure you encode your strings before sending them to a server and vice versa. `NSString`’s `stringByAddingPercentEscapesUsingEncoding` does not encode some characters such as ampersand and slashes. You can write your own method using Core Foundation’s percent escape functions. That gives you control over which characters to encode:

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
	
To test for empty strings, you must take into consideration that there might be whitespace and remove it: 
	
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
	
You should validate user input so that it's of the expected type and range. For example, if a user needs to enter an email address, you can check for a valid address:

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
	
Limit the number of characters allowed in a `UITextField` by setting it’s delegate and implementing the `shouldChangeCharactersInRange` method:

	-(BOOL) textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string
	{
	    NSUInteger newLength = [[textField text] length] + [string length] - range.length;
	    if (newLength > kMaxSearchLength)
	    {
	        return NO;
	    }
	
	    return YES;
	}
	
The delegate method to implement for a `UITextView` is this:

	- (BOOL)textView:(UITextView *)textView shouldChangeTextInRange:(NSRange)range replacementText:(NSString *)text;
	
### Images

In addition to text, you should check imported files. For example, for a JPEG file, the first two bytes and last two bytes are always **FF** **D8** and **FF** **D9**:
	
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
	
You should validate registered URL schemes that can send commands to the application. Make sure you don’t accept commands that are complete URL or path names or that get passed directly into your code. They could cause buffer overflows or format string attacks.

You'll need to sanitize user input an environment will execute, such as SQL or a server that will run code. Make sure to remove characters of the specific language that are susceptible to any command injection attacks. Common examples are **'** **"** **;** **%** **/** and **\**.

An often overlooked area of input is unarchiving object graph serializations or saved data objects (XIB and NIB files included). You should perform validation in `initWithCoder` and only serialize classes that conform to NSSecureCoding.

One more overlooked area is network callback functions and signals. Say you have a runloop that triggers an event when it receives data, such as a socket connection waiting for data. In addition to validating the data, don't assume in your program flow when you think the callback will fire. Be prepared for it to happen anytime and also be very careful validating the input received. Make sure the read buffer will not overflow, or block the main thread resulting in a denial of service attack launched from the server. POSIX Networking and CFNetworking are out of scope of this post, so see [the CFSocket article](https://kolinsturt.github.io/lessons/2013/01/01/CFSocket) for more details.

You learned secure coding practices for Objective-C. To learn about secure programming for other environments, see the Secure Coding in Swift and Null Safety Tutorial for Kotlin.
