---
layout: post
category : lessons
tagline: "app-provided passwords for keychain items"
title: Keychain Entropy on iOS 9
author: Kolin Stürt
description: Provide an application password for keychain items in iOS 9
tags : [iOS9, "iOS 9", keychain, entropy, "access control list", SecAccessControlCreateWithFlags, kSecAccessControlApplicationPassword, "Application Password", LACredentialTypeApplicationPassword, kSecAttrAccessControl, kSecUseAuthenticationContext, LAContext]
---
{% include JB/setup %}

## Keychain Entropy on iOS 9

This article assumes you're familiar with the keychain and it's query style. If that is new to you, check out [Keychain Services in C](https://kolinsturt.github.io/lessons/2012/05/01/KeychainServices) or the [Keychain in Swift](http://code.tutsplus.com/tutorials/securing-ios-data-at-rest--cms-28528?_ga=2.184040613.269493678.1591988254-1806496344.1591988254) tutorial.

iOS 9 added the ability to add entropy while encrypting items in the keychain. This involves using the Application Password option as part of an access control list. This opens up opportunities to provide entropy in different ways. For example, you could use PBKDF2 from user's provided passwords that let them save and load items from the keychain. You could authenticate with a server which securely returns a token for the keychain. While not very secure, you could hardcode an obfuscated password in the binary to prevent offline keychain dumps.

### Access Control

You can provide an access control object and context to the keychain query dictionary. The keys are `kSecUseAuthenticationContext` and `kSecAttrAccessControl`. 

`kSecUseAuthenticationContext` lets you pass in an authentication context. You'll create an `LAContext` that allows you to set an app-specific password:

	LAContext *localAuthenticationContext = [[LAContext alloc] init];
	NSData *theApplicationPassword = [@"1234" dataUsingEncoding:NSUTF8StringEncoding];
	            [localAuthenticationContext setCredential:theApplicationPassword type:LACredentialTypeApplicationPassword];

Next, use a `SecAccessControlRef` object that will contain access control conditions. Use the `SecAccessControlCreateWithFlags` function (available in iOS 8 +) to create this object:

	SecAccessControlRef accessControl = SecAccessControlCreateWithFlags(kCFAllocatorDefault, kSecAttrAccessibleAfterFirstUnlock, kSecAccessControlApplicationPassword, &error);
	
The parameters you passed are a default allocator, `kSecAttrAccessibleAfterFirstUnlock` as the `kSecAttrAccessible` protection class and kSecAccessControlApplicationPassword as the bitmask option - this denotes that an application provided password for the keychain item. Then you passed in an error object.

### Adding a Keychain Item

Now that you have a context and an access control object, add them to a keychain query dictionary:

	NSDictionary *saveDictionary = @{
	                            (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
	                            (__bridge id)kSecAttrService: @"testService2",
	                            (__bridge id)kSecAttrAccount: @"testAccount2",
	                            (__bridge id)kSecValueData: [@"qqqq" dataUsingEncoding:NSUTF8StringEncoding],
	                            (__bridge id)kSecAttrAccessControl: (__bridge id)accessControl,
	                            (__bridge id)kSecUseAuthenticationContext:localAuthenticationContext
	                            };

* You can use this with `SecItemAdd()`, `SecItemCopyMatching()` and other keychain query functions. 
* These additions are only available on iOS 9.
* Omitting `kSecAttrAccessControl`and `kSecUseAuthenticationContext` in `SecItemCopyMatching` when `SecItemAdd` included them will cause `SecItemCopyMatching` to block for a long period of time. Sometimes it returns **_BSMachError: (os/kern) invalid capability (20)**. Check for the existence of the instantiated objects and if they don't exist, return an error. 
* If you supply an incorrect application password, the system returns OSStatus error `-25293`.

### Putting It All Together

Here's an example that stores and retrieves a password from the keychain using the new application password option. If the user is not on iOS 9 then there should be a fallback at the last else statement to save and load without using the new feature.

Remember to add the LocalAuthentication framework to your project and import it in your code:

	#import <LocalAuthentication/LocalAuthentication.h>
	#import <Security/SecAccessControl.h>
	
Here's the full example:

	- (void)applicationPasswordExample
	{
	    if ([LAContext class]) // iOS 8 or later
	    {
	        LAContext *localAuthenticationContext = [[LAContext alloc] init];
	        if ([localAuthenticationContext respondsToSelector:@selector(setCredential:type:)]) //iOS 9 + , could use [[NSProcessInfo processInfo] operatingSystemVersion] instead
	        {
	            OSStatus status = noErr;
	            CFErrorRef error;
	            SecAccessControlRef accessControl = SecAccessControlCreateWithFlags(kCFAllocatorDefault, kSecAttrAccessibleAfterFirstUnlock, kSecAccessControlApplicationPassword, &error);
	            if (accessControl)
	            {
	                NSData *theApplicationPassword = [@"1234" dataUsingEncoding:NSUTF8StringEncoding];
	                [localAuthenticationContext setCredential:theApplicationPassword type:LACredentialTypeApplicationPassword];
	                
	                NSDictionary *saveDictionary = @{
	                                                 (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
	                                                 (__bridge id)kSecAttrService: @"testService",
	                                                 (__bridge id)kSecAttrAccount: @"testAccount",
	                                                 (__bridge id)kSecValueData: [@"qqqq" dataUsingEncoding:NSUTF8StringEncoding],
	                                                 (__bridge id)kSecAttrAccessControl: (__bridge id)accessControl,
	                                                 (__bridge id)kSecUseAuthenticationContext:localAuthenticationContext
	                                                 };
	                
	                status = SecItemAdd((__bridge CFDictionaryRef)saveDictionary, nil);
	                if (status == noErr)
	                {
	                    NSLog(@"Item stored to keychain");
	                }
	                
	                NSDictionary *loadDictionary = @{
	                                                 (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
	                                                 (__bridge id)kSecAttrService: @"testService",
	                                                 (__bridge id)kSecAttrAccount: @"testAccount",
	                                                 (__bridge id)kSecReturnData: @(YES),
	                                                 (__bridge id)kSecMatchLimit: (NSString *)kSecMatchLimitOne,
	                                                 (__bridge id)kSecAttrAccessControl: (__bridge id)accessControl,
	                                                 (__bridge id)kSecUseAuthenticationContext:localAuthenticationContext
	                                                 };
	                
	                CFDataRef passwordData = NULL;
	                status = SecItemCopyMatching((CFDictionaryRef)loadDictionary, (CFTypeRef *)&passwordData);
	                
	                if ( (status == noErr) && (0 < CFDataGetLength(passwordData)) )
	                {
	                    CFStringRef string = CFStringCreateWithBytes(kCFAllocatorDefault, CFDataGetBytePtr(passwordData), CFDataGetLength(passwordData), kCFStringEncodingUTF8, FALSE);
	                    if (string)
	                    {
	                        CFShow(string);
	                        CFRelease(string);
	                    }
	                }
	                
	                if (passwordData != NULL)
	                {
	                    CFRelease(passwordData);
	                }
	                
	                CFRelease(accessControl);
	            }
	            else
	            {
	                NSLog(@"Error creating access control object");
	            }
	        }
	        else
	        {
	            //Perform save without application password (without kSecAttrAccessControl and kSecUseAuthenticationContext) fields
	        }
	    }
	    else
	    {
	        //Perform save without application password (without kSecAttrAccessControl and kSecUseAuthenticationContext) fields
	    }
	}


To learn move about keychain capabilities, check out the [Keychain in Swift](http://code.tutsplus.com/tutorials/securing-ios-data-at-rest--cms-28528?_ga=2.184040613.269493678.1591988254-1806496344.1591988254) and [Keys and Credentials on Android](http://code.tutsplus.com/tutorials/keys-credentials-and-storage-on-android--cms-30827?_ga=2.254295179.269493678.1591988254-1806496344.1591988254) tutorial.
