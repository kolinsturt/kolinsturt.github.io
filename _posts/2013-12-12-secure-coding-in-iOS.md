---
layout: post
category : lessons
tagline: "Objective-C Runtime"
title: Objective-C Runtime Security and Obfuscation
description: Objective-C Runtime - Security and Obfuscation
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
redirect_from:
  - /lessons/2013/12/12/secure_coding_in_iOS
---
{% include JB/setup %}

### Objective-C Runtime Security and Obfuscation

Instance variables and methods in Objective-C classes are hardly secure. There are at least access specifiers that you can use, similar to C++:

* @private - instance variables are only visible in the class which they were declared.
* @protected (default) - instance variables are visible to the class and any subclass of the class.
* @package Often used in frameworks - an instance variable is globally visible in the binary but not in any part of the program that the framework is linked to. Note that on 32-bit Mac OS X programs, this acts the same as @public. 
* @public instance variables are visible everywhere throughout the application

However, this is not in any way secure encapsulation. If an object - `MyObject` has an instance variable `NSString *_secretString`, you can still access it even if it is private:

	MyObject *myObject = [[MyObject alloc] init];
	[myObject setValue:@"aha" forKey: @“_secretString”];

You accessed the instance variable by name using Key-Value-Coding. To prevent this scenario from happening, you can implement a method in our class:

	+ (BOOL)accessInstanceVariablesDirectly 
	{
	 	return NO;
	}

If this method returns NO for our class, trying to set _secretString will instead terminate the application.

Methods that belong to classes have even less security. You can imitate private methods in Objective-C by keeping the method only in your implementation file without any declaration, or add the declaration only inside the implementation file as a class extension

	@implementation MyObject ()
	- (void)_privateMethod;
	@end

While this informs users of the class that this method is for internal use only, it doesn't prevent users calling the method like this:

	    [myObject performSelector:@selector(_privateMethod) withObject:nil];

The only way around this is to set up a flag that prevents the method from being called while the application is in a particular state. You may not want the method called until certain other steps have been performed, for example, and you could track these steps using Boolean variables.

String constants can easily be dumped from the binary, but method names of classes can also be exposed in a decompiler such as [Ida](https://www.hex-rays.com/products/ida/index.shtml). So it may be important to rename sensitive methods to something mundane. A method that returns if the user is logged in might be one such method. One preventative measure you can take is to swap out a mundane method at runtime for another function. You can do this with the `class_replaceMethod()` Objective-C runtime function:

	#import <objc/runtime.h>

	class_replaceMethod(myObject,
                            @selector(mundaneMethod),
                            (IMP)TheRealWork,
                            "v@:");

	static void TheRealWork(id self, SEL _cmd)
	{
	    MyObject *myObject = (MyObject *)self;
	    //do some private work, or call other methods...
	}

Another way to make a method call appear as an innocent method is by making it appear to come from some other class by using a delegate with an invocation method call. You can make an innocent looking class in the header file:

	@interface InnocentButton : UIButton
	@end

and the implementation file:

	@implementation InnocentButton

	//obfuscated code
	-(CFStringRef)myString
	{
	    //some secret stuff here …
	    return CFSTR("here is a string");
	}
	
	@end

Then make a proxy object:

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
	
Now you can call the function on this object, except that it is secretly calling a different method on a different object:

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

If your obfuscation is aimed at integrity protection, check out the [Application Integrity](https://kolinsturt.github.io/lessons/2013/03/05/integrity) article for iOS and [Getting Started With ProGuard](https://www.raywenderlich.com/7449-getting-started-with-proguard) and [SafetyNet API documentation](https://developer.android.com/training/safetynet/attestation) for Android.
