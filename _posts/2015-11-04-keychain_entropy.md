---
layout: post
category : lessons
tagline: "app-provided entropy for keychain items"
title: iOS Keychain Item Entropy
author: Kolin Stürt
description: Providing entropy for keychain items in iOS 9
tags : [iOS9, "iOS 9", keychain, entropy, "access control list", SecAccessControlCreateWithFlags, kSecAccessControlApplicationPassword, "Application Password", LACredentialTypeApplicationPassword, kSecAttrAccessControl, kSecUseAuthenticationContext, LAContext]
---
{% include JB/setup %}

## iOS 9 Keychain Item Entropy

iOS 9 offers the ability to add entropy while encrypting items in the keychain. This involves using the Application Password option as part of an access control list. This opens up opportunities to provide entropy in different ways. For example, we could use PBKDF2 from a user provided password that lets them save and load items from the keychain. We could authenticate with a server which would securely return a key or password that could be used with the keychain, or while obviously less secure, we could hardcode an obfuscated password in the binary.

### Access Control

The keychain is accessed like always except that we add two new parameters; a context and an access control options object to the keychain query dictionary being used. The keys are kSecUseAuthenticationContext and kSecAttrAccessControl. (For more general information about how the keychain services work, see [here](https://collinbstuart.github.io/lessons/2012/05/01/KeychainServices/))

kSecUseAuthenticationContext lets you pass in an authentication context. We will create an LAContext that allows us to set an app-specific password.

	LAContext *localAuthenticationContext = [[LAContext alloc] init];
	NSData *theApplicationPassword = [@"1234" dataUsingEncoding:NSUTF8StringEncoding];
	            [localAuthenticationContext setCredential:theApplicationPassword type:LACredentialTypeApplicationPassword];

Next we use a SecAccessControlRef object that will contain access control conditions.
To create this object we use the SecAccessControlCreateWithFlags function (available in iOS 8 +) which takes the parameters of a default allocator, the kSecAttrAccessible protection class to use, an options bitmask and an error object. In our example we will use kSecAttrAccessibleAfterFirstUnlock as the protection class and kSecAccessControlApplicationPassword as the option - this denotes that an application provided password for encryption will be used.

	SecAccessControlRef accessControl = SecAccessControlCreateWithFlags(kCFAllocatorDefault, kSecAttrAccessibleAfterFirstUnlock, kSecAccessControlApplicationPassword, &error);

### Adding a Keychain Item

Now that we have a context and an access control object, we can add them to a regular keychain query dictionary, for example:

	NSDictionary *saveDictionary = @{
	                            (__bridge id)kSecClass: (__bridge id)kSecClassGenericPassword,
	                            (__bridge id)kSecAttrService: @"testService2",
	                            (__bridge id)kSecAttrAccount: @"testAccount2",
	                            (__bridge id)kSecValueData: [@"qqqq" dataUsingEncoding:NSUTF8StringEncoding],
	                            (__bridge id)kSecAttrAccessControl: (__bridge id)accessControl,
	                            (__bridge id)kSecUseAuthenticationContext:localAuthenticationContext
	                            };

and call SecItemAdd(), SecItemCopyMatching(), or others.

* These additions are only available on iOS 9, so you would want to only use the new options for storing keychain items that will be accessed on a single device that has iOS 9. 

* Not including kSecAttrAccessControl and kSecUseAuthenticationContext in the SecItemCopyMatching search query when they were used during SecItemAdd will cause SecItemCopyMatching to block for a long period of time (often returning  _BSMachError: (os/kern) invalid capability (20) ) . So check for the existence of the instantiated objects and if they do not exist, just return an error instead.

* If the incorrect application password is used to try and retrieve an item, an OSStatus error of -25293 (password incorrect) is returned.

### Putting It All Together

Here is a complete example that first stores and then retrieves a password from the keychain using the new application password options. If the user is not on iOS 9 then there should be a fallback at the last else statement to save and load without using the new features.

Remember to add the LocalAuthentication framework to your project and

	#import <LocalAuthentication/LocalAuthentication.h>
	#import <Security/SecAccessControl.h>
	
in the file you are working with.

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


Feel free to send me any [issues](https://github.com/CollinBStuart/CollinBStuart.github.io/issues) you may find. 

