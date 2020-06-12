---
layout: post
category : lessons
tagline: "iOS app hardening"
title: App hardening and secure programming
description: Secure programming and app hardening on iOS
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
---
{% include JB/setup %}

### Objective-C Runtime - Security and Obfuscation

As you may already know, instance variables and methods in Objective-C classes are hardly secure. There are at least access specifiers that we can use, similar to C++:

* @private - instance variables are only visible in the class which they were declared.
* @protected (default) - instance variables are visible to the class and any subclass of the class.
* @package Often used in frameworks - an instance variable is globally visible in the binary but not in any part of the program that the framework is linked to. Note that on 32-bit Mac OS X programs, this acts the same as @public. 
* @public instance variables are visible everywhere throughout the application

However, this is not in any way secure encapsulation. For example, if an object - MyObject has an instance variable, NSString *_secretString, we can still access it even if it is private

	MyObject *myObject = [[MyObject alloc] init];
	[myObject setValue:@"aha" forKey: @“_secretString”];

We accessed the instance variable by name using Key-Value-Coding. To prevent this scenario from happening however, we can implement a method in our class 


	+ (BOOL)accessInstanceVariablesDirectly 
	{
	 	return NO;
	}

If this method returns NO for our class, trying to set _secretString will instead terminate the application.

Methods that belong to classes have even less security. You can imitate private methods in Objective-C by keeping the method only in your implementation file without any declaration, or add the declaration only inside the implementation file as a class extension

	@implementation MyObject ()
	- (void)_privateMethod;
	@end

While this informs users of the class that this method is for internal use only, it doesn't prevent users calling the method like so

	    [myObject performSelector:@selector(_privateMethod) withObject:nil];

The only way around this is to set up some kind of flag that prevents the method from being called while the application is in a particular state. You may not want the method being called until certain other steps have been performed, for example, and you could track these steps using Boolean variables.

String constants can easily be dumped from the binary, but method names of classes can also be exposed in a decompiler such as [Ida](https://www.hex-rays.com/products/ida/index.shtml). So it may be important to rename sensitive methods to something mundane. A method that returns if the user is logged in might be one such method. One preventative measure you can take is to swap out a mundane method at runtime for another function. You can do this with the class_replaceMethod() Objective-C runtime function.

	#import <objc/runtime.h>

	class_replaceMethod(myObject,
                            @selector(mundaneMethod),
                            (IMP)RealDoWork,
                            "v@:");

	static void RealDoWork(id self, SEL _cmd)
	{
	    MyObject *myObject = (MyObject *)self;
	    //here we do some private work, or call other methods...
	}

Another way to make a method call appear as an innocent method is by making it appear to come from some other class by using a delegate with an invocation method call. We can make an innocent looking class, in the header file

	@interface InnocentButton : UIButton
	@end

and the implementation file

	@implementation InnocentButton

	//obfuscated code
	-(CFStringRef)myString
	{
	    //some secret stuff here …
	    return CFSTR("here is a string");
	}
	
	@end

We can make a proxy object

	@interface P : NSObject
	@property (nonatomic, assign) id delegate;
	@end
	@implementation P
	
	//get message signature for messages coming in
	- (NSMethodSignature *)methodSignatureForSelector: (SEL) aSelector
	{
		if ([[self class] instancesRespondToSelector:aSelector])
	    {
			return [[self class] instanceMethodSignatureForSelector:aSelector];
		}
		return [_delegate methodSignatureForSelector:aSelector];
	}
	
	//forward message to another object
	- (void)forwardInvocation: (NSInvocation *)anInv
	{
		[anInv invokeWithTarget:_delegate];
	}
	@end
	
Now we can call the function on this object, except that it is secretly calling a different method on a different object

		InnocentButton *innocentButton = [[InnocentButton alloc] init];

		P *p = [[P alloc] init];
        [p setDelegate:innocentButton];
        
        SEL myStringSelector = sel_registerName("myString");
        CFStringRef string = (CFStringRef)[p performSelector:myStringSelector];
        [p setDelegate:nil];
        [p release];

While none of these methods are foolproof, they can help obfuscate any sensitive code. This is not the only way to hide sensitive information. Objective-C runtime features require the binary to store a lot of class information so that we can pretty much rebuild the entire class interface from the binary. Someone can use 'strings', [nm](http://linux.die.net/man/1/nm), or [otool](http://www.manpagez.com/man/1/otool/) to do a strings dump of the binary as mentioned above, or to list all the classes and methods of a binary. Therefore, any sensitive methods could instead be written using C functions instead of Objective-C methods. That way the names of methods will not be revealed in the dump.

If you have a method that performs some sensitive operation, such as enabling paid features in your app or checking authorization, it may make sense to try and turn the method into a C inline function. The reason is that with inline functions, if the function gets called from many places in your code, the actual code is pasted into the calling function each time the function call is made. This prevents against simply hijacking one method in your application to gain access as the attacker would need to hijack each place or time the function was called. Additionally, making functions inline may help obfuscate the code when viewed in a debugger breakpoint as it may be hidden in assembly.


	static inline void SensitiveFunction() __attribute__((always_inline));


In order to hint inline functions under GCC, you can add the -Winline and -finline-functions flags in the "Compiler Flags" section of XCode. Go to the project settings and choose the "Build Phases" tab. Expand the section for the "Compile Sources" and double click on any file under the "Compiler Flags" section to bring up a box to enter the flag.

Remember that the inline keyword is only a hint to the compiler. It is not guaranteed that the function will be called inline.


### End 

Some testing and trial and error are needed and this is definitely true with the XCode optimizer, which can either expose part of the code, or in most cases it can help obfuscate your code during disassembly. Apple's optimizer tries to change up the code in order to speed up routines or to decrease the size of the compiled code. In your project settings, select the "Build Settings" tab, and search for "optimization". You should see an Optimization Level field. Setting the optimization level to Fastest -O3 may help obfuscate the code. In some cases it can make it easier to disassemble. Trial and error is needed (for example by using otool). For more information about this setting, see the [Compiler-Level Optimizations](https://developer.apple.com/library/mac/documentation/General/Conceptual/MOSXAppProgrammingGuide/Performance/Performance.html#//apple_ref/doc/uid/TP40010543-CH9-102307) section of the Apple reference guide.

**Note: Make sure to test your app using the optimization level you would like to use for production. XCode sets the optimization level differently for the release scheme of your app than for the debug scheme. Change the "Optimization Level" for debug in the Code Generation section of the build settings, or go to Product -> Scheme -> Edit Scheme and in the info tab set the Build Configuration to release. Make sure you app works correctly in this setting before it reaches the public**

Since we are settings XCode settings, lets move on to discuss other importing XCode project settings

## XCode Project Settings

Other XCode project settings can help secure against vulnerabilities. 

If you are using GCC, you can add the _FORTIFY_SOURCE Preprocessor Macro which checks for particular buffer overflows. You can find an in-depth description of what overflows it prevents [here](http://gcc.gnu.org/ml/gcc-patches/2004-09/msg02055.html)

### Stack Smashing Protection

SSP checks for buffer overflows, most importantly for stack smashing attacks. It adds a guard variable when entering a function that contains vulnerable objects such as buffers. Then when the function exits, the variable is checked. 

You can enable SSP in XCode by adding the -fstack-protector-all flag in the Compiler Flags section of XCode. Go to the project settings and choose the "Build Phases" tab. Expand the section for the "Compile Sources" and double click on any file under the "Compiler Flags" section to bring up a textbox to enter the flag.

TIP: You can hold shift to highlight multiple entries and hit enter to open a textbox to change the compiler flags for all selected entries. However, any previous flags will be overwritten with the new text.

### Address Space Randomization
Position independent code can be placed in primary memory and will execute without problems regardless of what their absolute address is. Different locations are used for the stack, heap, and executable code for the application each time it launches. This complicates performing buffer overflows as it is not easily possible to know where buffers are in memory and where code is specifically located. To explicitly turn this feature on, set the -pie flag in the "Other Linker Flags" area of XCode's "Build Settings".


### Non-Executable Stack and Heap
Newer processors can mark parts of memory as non-executable. This is set by something called an NX bit. Both iOS and OS X use this feature by default.

### Code Analysis
Last but not least, be sure to always run the static analyzer on your code, as it checks for security issues such as format string attacks and functions that are susceptible to buffer overflows. You can run the static analyzer from the Product menu in XCode. Different security analyzation options can be set by selecting the project settings in XCode, navigation to the "Build Settings" tab. In the search textbox, type Security. A group will appear called "Static Analyzer - Issues - Security". From here you can turn on checking of specific issues such as "Use of 'rand' functions"  (which has [vulnerabilities](http://www.tech-faq.com/random-number-vulnerability.html)) instead of the newer arc4rand() or SecRandomCopyBytes() functions.

Now that we covered XCode settings to help prevent buffer overflows, lets move on to how to prevent buffer overflows in code.

##Buffer Overflows

In Apple's [Secure Programming Guide](https://developer.apple.com/library/mac/documentation/security/conceptual/SecureCodingGuide/Articles/SecurityGuidelines.html#//apple_ref/doc/uid/TP40009511-SW1), there is a list of common functions susceptible to buffer overflow attacks, and which ones to use in replacement. 

* strcat -> strlcat
* strcpy -> strlcpy
* strncat -> strlcat
* strncpy -> strlcpy
* sprintf -> snprintf (see note) or asprintf
* vsprintf -> vsnprintf (see note) or vasprintf
* gets -> fgets (see note) or use Core Foundation or Foundation APIs

Apple states in their document


> Security Note for snprintf and vsnprintf: The functions snprintf, vsnprintf, and variants are 	dangerous if used incorrectly. Although they do behave functionally like strlcat and similar in 	that they limit the bytes written to n-1, the length returned by these functions is the length 	that would have been printed if n were infinite.
	
> For this reason, you must not use this return value to determine where to null-terminate the string or to determine how many bytes to copy from the string at a later time.
	
> Security Note for fgets: Although the fgets function provides the ability to read a limited amount of data, you must be careful when using it. Like the other functions in the “safer” column, fgets always terminates the string. However, unlike the other functions in that column, it takes a maximum number of bytes to read, not a buffer size.
	
> In practical terms, this means that you must always pass a size value that is one fewer than the size of the buffer to leave room for the null termination. If you do not, the fgets function will dutifully terminate the string past the end of your buffer, potentially overwriting whatever byte of data follows it.

If you are writing code in C, you can use the Core Foundation representation of a string, CFStringRef. All of it's string manipulation functions start with CFString and are safe against buffer overflows.

If you are using Foundation, the same is true for the NSString class. NSString and CFString are also toll-free bridged in that they can be casted back and forth between the two.

If you are porting code in C++, std::string is safe from buffer overflows.

If you must use C, here are some examples to help avoid buffer overflows.

Try not to use hard coded buffer sizes such as 

	UInt8 buffer[512];
	if (size <= 511) 
	{
	   ;
	}


It would be better to define the buffer size once

	#define BUFFER_SIZE 512
	UInt8 buffer[BUFFER_SIZE];
	if (size < BUFFER_SIZE) 
	{
	   ;
	}
	
	
It is even better to do this

	if (size < sizeof(buffer)) 
	{
	   ;
	}


### Integer Overflows

An important point about checking for integer overflows is to not perform any checks that are not being used later in code. If the compiler sees checks where the result is not being used, it may just optimize the test right out of the code. For example, we can check the result of a potentially overflowing multiplication

size_t byteSize = (first * second);
if ( (bytes < first) || (bytes < second) ) //check for overflow
{
	;
}

Apple suggests in their guide

>the only correct way to test for integer overflow is to divide the maximum allowable result by the multiplier and comparing the result to the multiplicand or vice-versa. If the result is smaller than the multiplicand, the product of those two values would cause an integer overflow.

Here is an example

	if (first > 0 && second > 0 && SIZE_MAX / first >= second) 
	{
	    size_t byteSize = (first * second);
	    //...make allocation
	}



### Buffer Underflows

In order to prevent buffer underflows, make sure that you zero all buffers before you begin to use them. Do this with memset(), bzero(), or calloc() for example. When performing an alloc or init function, check that the function returned a success result code before processing the resulting data. If applicable, use any value returned by a function that tells you the size of the data returned and use this value rather than a hard-coded one. If you have a constant of what the size should have been, you can check this and fail gracefully if you did not receive the expected size instead of going on to process the data. The same should apply to any functions that operate on your data. For example, make sure a write call completes successfully before operating on that data. The same is said for opening and reading data. Make sure the size and type of data is of what is expected. Object strings such as NSString, CFStringRef or std::string handle size checks themselves, so it is best to leave the string in their object form if possible. Converting objects to primitive C types can cause vulnerabilities. CFString's CFStringCreateWithCString() for example does not take into account the length of the C string buffer that will be used.  Also, if possible, it's best to separate buffer and data operations from string manipulation functions.

CFDataRef lets you access the underlying primitive data (CFDataGetBytes) in which it's memory was managed my malloc and free as opposed to the reference counting scheme. Therefor, there is the potential for the buffer to be freed and be overwritten or reused accidentally.

Collection classes such as CFSet, CFArray, and CFDictionary fortunately create objects so that the user must first pass in the number of objects in the collection. Make sure this number is correct. Some of foundation's methods let you pass in a list of objects of a variable length with nil as the last element. If you forget to pass in nil, the initialize will continue reading into the process's stack. A warning will be issued in XCode if you forget the nil at the end, so it is a good idea to pay attention to warnings, eliminate all warnings so that new ones will stand out, and treat warnings as errors. 

CFArrayGetValues (or NSArray getObjects) will create a primitive C array but it does not check the length of the buffer being passed in. If the buffer is not big enough it will overflow.

### Validation

NSFileHandle is vulnerable to race conditions. It's method fileHandleForWritingAtPath returns an object when there is already a file at the path. Creating and opening the file happens in two separate steps so that a malicious user can change the created file with other data before the file is read. For example, the file can be changed to link to another file so that when the application goes to write to the file, it is instead overwriting an unknown file. It is better to create a temporary file with mktemp() and use CFFileDescriptor's CFFileDescriptorCreate, or NSFileHandle's initWithfileDescriptor.

	fd = mkstemp(tmpfile); // check for -1 error
	NSFileHandle *myhandle = [[NSFileHandle alloc] initWithFileDescriptor:fd];
	
Sanitize and validate files coming from any location outside the app sandbox. In addition to checking sizes, it is important, especially when receiving data from a server, to check that it is the correct type. Make sure to always check for NULL. Most CoreFoundation functions will crash if you try operating on NULL. For example, CFRelease will crash if it is passed NULL so make sure arrays, dictionaries and other created objects have successfully been created before releasing.

	if (array)
	{
		CFRelease(array);
		array = NULL;
	}
	
If you assign your ivars to NULL or nil after use, then you can safely check if an object exists as in the above example. If you do not set ivars to NULL, they may be reused after releasing and may point to some unknown object. The only side effect of settings all ivars to NULL is that it may hide if there is a problem with the logic of the code. A crash in development mode is good in that it draws attention to an underlying problem.

Each Core Foundation object has a type ID. While there isn't a specific list of types that should be checked against, to check for a specific type of object, use the CFGetTypeID() on the object in question and compare it with the object you want to check for. For example, if you want to make sure the object is a string, use CFStringGetTypeID. Each Core foundation object has a CFxGetTypeID function. To check for CFNullRef's do this

	if ( CFGetTypeID(data) != CFNullGetTypeID() )

In Foundation, you can do this

	if ([someObject isKindOfClass: [NSArray class]]) 

Additionally, you can see if an object conforms to a protocol before trying to send messages to it

	if ([someObject conformsToProtocol: @protocol(MyProtocol)]) 

And before you send a delegate a method, make sure the app will not crash on "unrecognized selector"

	if ([someObject respondsToSelector:@selector(someMethod)])

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
	
	
	
Another point of entry that should be validated is any registered URL scheme that can send commands to the application. Make sure you don't accept commands that are complete URL or pathnames, or that directly get passed into your code that could cause buffer overflows or format string attacks.

Be extremely careful when any user input is passed into commands that will be executed by an SQL server or server that will run code. Make sure to remove any characters of the specific language that the server will be processing so that the input is not susceptible to any command injection attacks.

Another often overlooked area that should be validated is from unarchiving object graph serializations, or saved data objects (XIB and NIB files included). If the archived data is not stored or transferred securely, either data or even method implementations can be changed. Validation should be done in initWithCoder and only save or package classes that conform to NSSecureCoding. 

One more overlooked area is network callback functions and signals. If you are setting up a runloop so that an event will be triggered when data is received; such as a socket connection waiting for server data, you can never be sure that the server process hasn't been compromised. Therefor, do not assume in your program flow when you think the callback will be fired. Be prepared for it to happen anytime and also be very careful validating the input received. Make sure the read buffer will not overflow, or block the main thread resulting in a denial of service attack that was successfully launched from the server. POSIX Networking and CFNetworking are out of scope of this post, so see [this guide](http://collinbstuart.github.io/lessons/2013/01/01/CFSocket) for more details.

### Format String Attacks

Any format string functions are susceptible to format string attacks, such as printf, CFStringCreateWithFormat, or NSString's stringWithFormat. To guard against these attacks, we just have to make sure that data is not passed as part of the format string. You may need to use your own format string, for example

	
	printf(buffer)
	
should be changed to
	
	printf("%s", buffer)
	
	 
	 

### Application Integrity

In addition to validating the integrity of data, an application's behaviour can deliberately be changed if the application detects the integrity of the environment has been compromised. A malicious user might have attached a debugger to the process to watch the contents of variables, or a mobile device may have been jailbroken. 

In order to check for jailbroken devices, we could look for the most common jailbreak applications that are installed 

	static const char* jailbreakApps[] = 
     {
         "/Applications/Cydia.app",
         "/Applications/limera1n.app",
         "/Applications/greenpois0n.app",
         "/Applications/blackra1n.app",
         "/Applications/blacksn0w.app",
         "/Applications/redsn0w.app",
         NULL,
     };
     
We can use NSFileManager's fileExistsAtPath, or we can search specifically for a file using the stat() command

	Boolean isJB = FALSE;
    struct stat theStat;
    int result = (stat("/Applications/Cydia.app", &theStat) == 0);
    if (result)
    {
        isJB = TRUE;
    }
    
Most jailbreak patches install the Mobile Substrate dynamic library. This is a common library that allows programmers to make runtime patches to system functions. This can be dangerous if your application is relying on system functions. We can look for this library to try to determine if a device is jailbroken

	result = (stat("/Library/MobileSubstrate/MobileSubstrate.dylib", &theStat) == 0);
    if (result)
    {
        isJB = TRUE;
    }
    
Most redsn0w jailbreaks move the application folder from the small readonly partition to the large one on the device and symbolic link it, so we can check for this using the link version of the stat function, lstat()

	if (lstat("/Applications", &theStat) != 0)
    {
        if (theStat.st_mode & S_IFLNK) //mode of the file is a symbolic link
        {
            isJB = TRUE;
        }
    }
    
[Johnathan Zdziarski](http://www.zdziarski.com/blog/?cat=8) had researched that the size of the fstab in iOS (so far) has always been 80 bytes but noticed that on jailbreak devices, fstab is 65 bytes because it has been patched for read-write access

	stat("/etc/fstab", &theStat);
    off_t fstabSize = theStat.st_size;
    if (fstabSize < 80)
    {
        ; //possibly jailbroken
    }
    
He also mentions that many functions like the fork() function to fork a process are turned off for the app sandbox environment of iOS, therefor is fork() succeeds, the phone may be jailbroken. This check may not be able to be used unless you are writing adhoc code that does not need to be approved through the App store as you don't want to fork() a process. If the fork process succeeds, you'd need to exit the process with exit(0), however explicitly exiting an app is against Apple's review guidelines and may get your app rejected. If your using this in a test environment, make sure that you exclude when the user is running an iPhone simulator as you will incorrectly be notified of a jailbreak when their isn't one

	#if !(TARGET_IPHONE_SIMULATOR)
	    pid_t theID = fork();
	    if (theID >= 0) //if fork fails, it returns a negative number
	    {
	        isJB = TRUE;
	        //since this succeeds, we should exit(0) the forked process, but Apple will reject us for that code so it is not safe to use
	    }
	#endif


Last but not least, there are a few things we can try to do to detect if [a debugger is attached](https://developer.apple.com/library/mac/qa/qa1361/_index.html). The kernel has a flag, P_TRACED, that is set when a debugger is attached and we can simple check for it


	static inline Boolean IsDebuggerAttached()
	{
	    //setup local vars - kernel info structure
	    size_t size = sizeof(struct kinfo_proc);
	    struct kinfo_proc info;
	    int returnCode;
	    int name[4];
	    
	    //load in the kernel process info
	    memset(&info, 0, sizeof(struct kinfo_proc));
	    name[0] = CTL_KERN; //high kernel
	    name[1] = KERN_PROC; //process entries
	    name[2] = KERN_PROC_PID; //process id
	    name[3] = getpid(); //get our pid
	    
	    
	    returnCode = (sysctl(name, 4, &info, &size, NULL, 0));
	    
	    if (returnCode)
	    {
	        //some error
	        return FALSE;
	    }
	    
	    //the kernel sets P_TRACED flag when a debugger is attached and we can check for this
	    Boolean isDebuggerRunning = (info.kp_proc.p_flag & P_TRACED) ? TRUE : FALSE;
	    
	    return isDebuggerRunning;
	    
	}

Another thing we can do is block the application from running if a debugger is attached in production environments. Apple created a flag, PT_DENY_ATTACH for Mac OS X that could be used for your application to request that it not be traced or debugged. On iOS it doesn't exist, but it's value is 31 so we can just define it in our code. We use the ptrace system call to pass in a request integer to the first parameter - PT_DENY_ATTACH. You will obviously want to have your own flag to turn this test off for debug environments and on for production environments. Putting this function before the main run loop is set up will prevent the application from launching if a debugger is attached. 

	#import <dlfcn.h>
	#import <sys/types.h>
	
	void DisableDebugging(void);
	typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
	#if !defined(PT_DENY_ATTACH)
	#define PT_DENY_ATTACH 31
	#endif  // !defined(PT_DENY_ATTACH)
	
	void DisableDebugging(void)
	{
	    //dynamic linking - access object file handle
	    void *handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW); //modes are symbol lookup | get relocations
	    
	    //obtain the address of the ptrace symbol - that's what a debugger will be using
	    ptrace_ptr_t ptrace_ptr = dlsym(handle, "ptrace");
	    
	    //taken from header file for OS X - PT_DENY_ATTACH flag prevents a debugger from attaching to the calling process
	    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
	    
	    //close object
	    dlclose(handle);
	}
	
	int main(int argc, char *argv[])
	{
	
	    @autoreleasepool
	    {
	        DisableDebugging();
	
	        return UIApplicationMain( argc, argv, nil, NSStringFromClass([AppDelegate class]) );
	    }
	    
	}

### UIKit

The nice animation that happens when putting an app into background is achieved by iOS taking a screenshot of the app which it then uses for the animation. This image gets stored onto the phone. When you look at the list of open apps on iOS, this screenshot will be revealed as well. Make sure you hide any user sensitive data so that it does not get captured by the screenshot. To do this, set up a notification when the application is going into background to hide the UI with the setHidden:YES method for the UI elements you want to exclude. Then when coming to foreground you can unhide them. To set up notifications in foundation, do this

	 [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(gotoBackground) name:UIApplicationDidEnterBackgroundNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(comebackfromBackground) name:UIApplicationWillEnterForegroundNotification object:nil];
    
and to remove the notifications when the view disappears
    
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationDidEnterBackgroundNotification object:nil];
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationWillEnterForegroundNotification object:nil];
    
Or in Core Foundation
    
    CFNotificationSuspensionBehavior suspendBehavior = CFNotificationSuspensionBehaviorDeliverImmediately;
    CFNotificationCenterRef center = CFNotificationCenterGetLocalCenter();
    CFNotificationCenterAddObserver(center, (__bridge const void *)(self), BackgroundNotificationCallback, (CFStringRef)UIApplicationDidEnterBackgroundNotification, NULL, suspendBehavior);
    
Cleanup
    
    CFNotificationCenterRemoveObserver(CFNotificationCenterGetLocalCenter(), (__bridge const void *)(self), (CFStringRef)UIApplicationDidEnterBackgroundNotification, NULL);

Any sensitive information entered into text fields should use 
	
	UITextField *textField;
    [textField setSecureTextEntry:YES];

Additionally, non secure text fields by default have auto-correct. Some text that the user types, along with newly learned words are stored in a cache so that it is possible to retrieve some of the text that the user has previously entered in your application. To prevent this, turn off the autocorrect option as follows
 
	[myTextField setAutocorrectionType:UITextAutocorrectionTypeNo];

### File Protection

When you are saving files on a device, make sure to encrypt the user's data. For example, when you set up a Core Data Model, it can be stored using encryption by taking advantage of the NSFileProtectionComplete constant

	NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"Model.sqlite"];
    
    // Make sure the database is encrypted when the device is locked
    if (![[NSFileManager defaultManager] setAttributes:@{NSFileProtectionKey : NSFileProtectionComplete} ofItemAtPath:[storeURL path] error:NULL])
    {
        ;
    }
    
    
When you are creating files and directories, you can do the same
 
	BOOL ok = [[NSFileManager defaultManager] createFileAtPath:someFile contents:nil attributes:@{NSFileProtectionKey : NSFileProtectionComplete}];

    
    if (![[NSFileManager defaultManager] fileExistsAtPath:somePath])
    {
        [[NSFileManager defaultManager] createDirectoryAtPath:somePath withIntermediateDirectories:YES attributes:@{NSFileProtectionKey : NSFileProtectionComplete} error:nil];
    }

NSData has a method that can write it's data to a file. You can also add encryption at this point

	NSData *data;

	[data writeToFile:path options:(NSDataWritingAtomic | NSDataWritingFileProtectionComplete) error:&error];

You can enable data protection across your app with one setting by enabling data protection for the app id in the Apple provisioning portal website. Then in XCode's Project Settings, select the "Capabilities" tab and turn on "Data Protection"

Keep in mind that this type of data protection requires that the user has a passcode set on the device. If you would like to have control over when you apply your encryption, see [AES Encryption using Common Crypto](http://collinbstuart.github.io/lessons/2014/01/01/common_crypto)


### Logging

Log or audit both successful and failed connection attempts. It's common to log failed connection attempt as we often think it could be a malicious user trying passwords, for example. However, it's a good idea to log the successful attempts to make sure an intrusion doesn't happen without notice.

Make sure you do not log user sensitive data. Encrypt and store the log in a secure place. Here is an example of overriding the default functionality of NSLog to provide more functionality. Here we get the line number and function that the log was called from.

	#define NSLog(args...) _CFLog(@"DEBUG ", __FILE__,__LINE__,__PRETTY_FUNCTION__,args);

	void _CFLog(NSString *prefix, const char *file, int lineNumber, const char *funcName, NSString *format,...)
	{
	    //local variables, setup current date and time, arguments and message
	    va_list ap;
	    va_start(ap, format);
	    const char *formatString = "%s %s%50s:%3d - %s";
	    format = [format stringByAppendingString:@"\n"];
	    NSString *dateString = [[self dateFormatter] stringFromDate:[NSDate date]];
	    NSString *msg = [[NSString alloc] initWithFormat:[NSString stringWithFormat:@"%@",format] arguments:ap];
	    va_end (ap);
	    
	    //write to console file if enabled
	#ifdef ENABLE_LOGGING_TO_CONSOLE
	    fprintf(stderr, formatString, [dateString UTF8String], [prefix UTF8String], funcName, lineNumber, [msg UTF8String]);
	#endif
	    
	    //save log in cache
	    if (. . .validate the string)
	    {
	        char *charBuffer = NULL;
	        size_t size;
	        size = snprintf(NULL, 0, formatString, [dateString UTF8String], [prefix UTF8String], funcName, lineNumber, [msg UTF8String]);
	        charBuffer = (char *)malloc(size + 1);
	        if (charBuffer)
	        {
	            snprintf(charBuffer, size + 1, formatString, [dateString UTF8String], [prefix UTF8String], funcName, lineNumber, [msg UTF8String]);
	            // -> Add code to save or send log here
	        }
	        free(charBuffer);
	    }
	}
	
If a server is down, make sure you rate limit the frequency and number of logs sent, network retries and error messages. Make sure the user can cancel the process so that a denial of service attack is less possible.
	
### Conclusion

This concludes the post on iOS Security. For further information on security technologies, see

[Keychain Services](http://collinbstuart.github.io/lessons/2012/05/01/KeychainServices)

[Hashing Algorithms in Core Foundation](http://collinbstuart.github.io/lessons/2013/05/01/hashing_algorithms_in_core_foundation)

[AES Encryption using Common Crypto](http://collinbstuart.github.io/lessons/2014/01/01/common_crypto)

[Secure Wipe Memory](http://collinbstuart.github.io/lessons/2013/10/01/secure_wipe_memory)
