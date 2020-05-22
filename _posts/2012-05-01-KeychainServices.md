---
layout: post
category : lessons
tagline: "Keychain Services in Core Foundation"
title: iOS keychain in Core Foundation
description: Using Apple's keychain services API
author: Kolin Stürt
tags : [Keychain Services, SecItem, ios, Core Foundation, tutorial]
---
{% include JB/setup %}

## Keychain Services

### Storing passwords

There are many Foundation wrappers for the keychain services, [SSKeychain](https://github.com/soffes/sskeychain), for example. If you are working in a Core Foundation only environment, we will create convenience functions in this post as a way to understand how to use the keychain services.

We can think of the keychain as a typical database where we run queries on a table. (In fact, it is an SQLite Database, stored on the iPhone at /private/var/Keychains/keychain-2.db and /users/USERNAME/Library/Keychains/ on OS X). The functions of the keychain services API all take in a CFDictionary object which specifies the keychain identifiers and attributes of each function. For example, if we are storing a password, we can set the keychain class, kSecClass, to kSecClassGenericPassword. Each entry in the keychain has a service name. The "service name" is just an identifier; a "key" for whatever "value" we want to store in the keychain. To make sure that our keychain items get stored for our app only, and to prevent duplicate entries in the keychain, we also need to specify an account name. The "account name" would be an identifier for our app. For example, we could use a reverse DNS name to avoid conflicts such as "com.testcompany.keychaintest". Because each function takes a dictionary with all these parameters to make a query, it makes sense to avoid duplicate code; to keep this code all in once place so when we need to change any information, we only need to change it once. Lets make a query function that returns a CFDictionary now

	CFMutableDictionaryRef _QueryDictionaryForService(CFStringRef serviceString, CFStringRef accountString)
	{
	    //return a dictionary containing the key-value pairs for use with adding to keychain services
	    CFMutableDictionaryRef mutableDictionary = CFDictionaryCreateMutable(kCFAllocatorDefault, 0, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
	    CFDictionaryAddValue(mutableDictionary, kSecClass, kSecClassGenericPassword);
	    CFDictionaryAddValue(mutableDictionary, kSecAttrAccount, accountString);
	    CFDictionaryAddValue(mutableDictionary, kSecAttrService, serviceString);
	    CFDictionaryAddValue(mutableDictionary, kSecAttrAccessible, kSecAttrAccessibleWhenUnlocked); // accessible only when unlocked
	    
		return (CFMutableDictionaryRef)CFAutorelease(mutableDictionary);
	}

Note that we are adding an extra layer of security to our query dictionary so that our keychain items are only accessible when the user unlocks their device with a passcode. The kSecAttrAccessible key defines this behaviour, as we pass it kSecAttrAccessibleWhenUnlocked.

In a previous post, we talked about how to create a message digest or hash from a string. Generally, it is not advisable to store a password directly to storage. In the event that the keychain is compromised, the raw password will be revealed. In situations where the user will be entering the sensitive information repeatedly to gain access, such as a password, it's best to store the hashed value in the keychain and only compare the hashes during access control.

**This also brings up an important point to think about when relying on "the system". What happens when everyone relies on just one security architecture, and then it gets compromised. The security of our private information is at the mercy of companies and the government. When a security architecture is compromised, all the software applications that rely on it are also compromised.**

In order to get started, lets set up some globals for error handling, taking an idea from the SSKeychain Foundation helper class, Sam Soffes's error handling added to the enum of errors from SecBase.h, BadArguments, and also NoPassword. Lets define these now:

	/** string domain for keychain */
	CFStringRef kKeychainServiceErrorDomain = CFSTR("com.testproject.keychain");
	
	/** OSStatus Error codes from SecBase.h with the addition of a few error codes, inspired by SSKeychain */
	CF_ENUM(OSStatus, KeychainErrorCode)
	{
	    /** No error. */
	    KeychainErrorNone = errSecSuccess,
	    
	    /** Some of the arguments were invalid. */
	    KeychainErrorBadArguments = -1001,
	    
	    /** There was no password. */
	    KeychainErrorNoPassword = -1002,
	    
	    /** One or more parameters passed internally were not valid. */
	    KeychainErrorInvalidParameter = errSecParam,
	    
	    /** Failed to allocate memory. */
	    KeychainErrorFailedToAllocated = errSecAllocate,
	    
	    /** No trust results are available. */
	    KeychainErrorNotAvailable = errSecNotAvailable,
	    
	    /** Authorization/Authentication failed. */
	    KeychainErrorAuthorizationFailed = errSecAuthFailed,
	    
	    /** The item already exists. */
	    KeychainErrorDuplicatedItem = errSecDuplicateItem,
	    
	    /** The item cannot be found.*/
	    KeychainErrorNotFound = errSecItemNotFound,
	    
	    /** Interaction with the Security Server is not allowed. */
	    KeychainErrorInteractionNotAllowed = errSecInteractionNotAllowed,
	    
	    /** Unable to decode the provided data. */
	    KeychainErrorFailedToDecode = errSecDecode
	};



Now lets store a string in the keychain. In order to do his, we simply use the SecItemAdd() function which adds items to a keychain. We change our string we want to store into a CFData object and pass it in as a parameter to the query dictionary.

	CFDataRef passwordData = CFStringCreateExternalRepresentation(kCFAllocatorDefault, passwordString, kCFStringEncodingUTF8, 0);
    CFDictionarySetValue(keychainQueryDictionary, kSecValueData, passwordData);
    SecItemAdd(keychainQueryDictionary, NULL);
    
The whole helper function with error checking can look like this

	Boolean KeychainSetPassword(CFStringRef passwordString, CFStringRef serviceString, CFStringRef accountString, CFErrorRef *error)
	{
		OSStatus status = KeychainErrorBadArguments;
		if (0 < CFStringGetLength(serviceString) && 0 < CFStringGetLength(accountString))
	    {
			KeychainDeletePasswordForKeychainItem(serviceString, accountString, NULL);
			if (0 < CFStringGetLength(passwordString))
	        {
				CFMutableDictionaryRef keychainQueryDictionary = _QueryDictionaryForService(serviceString, accountString);
				CFDataRef passwordData = CFStringCreateExternalRepresentation(kCFAllocatorDefault, passwordString, kCFStringEncodingUTF8, 0);
	            CFDictionarySetValue(keychainQueryDictionary, kSecValueData, passwordData);
				status = SecItemAdd(keychainQueryDictionary, NULL);
	            CFRelease(passwordData);
			}
		}
		
		if (status != noErr && error != NULL)
	    {
			*error = CFErrorCreate(kCFAllocatorDefault, kKeychainServiceErrorDomain, status, NULL);
		}
		
		return status == noErr;
	}
	
#### Preventing Duplicates
	
In the above code, to prevent duplicate inserts we first delete the previous entry if there is one. Lets write that function now to delete entries. To do that we use the SecItemDelete() function

	Boolean KeychainDeletePasswordForKeychainItem(CFStringRef serviceString, CFStringRef accountString, CFErrorRef *error)
	{
		OSStatus status = KeychainErrorBadArguments;
		if (0 < CFStringGetLength(serviceString) && 0 < CFStringGetLength(accountString))
	    {
			CFMutableDictionaryRef keychainQueryDictionary = _QueryDictionaryForService(serviceString, accountString);
			status = SecItemDelete(keychainQueryDictionary);
		}
		
		if (status != noErr && error != NULL)
	    {
			*error = CFErrorCreate(kCFAllocatorDefault, kKeychainServiceErrorDomain, status, NULL);
		}
		
		return status == noErr;
	}	
	
### Retrieving Passwords

Now all that's left is to retrieve an entry from the keychain. We will do this using the SecItemCopyMatching() function. This will return a base object of CFTypeRef, in our case this will be a CFStringRef. If you are familiar with foundation, you can think of CFTypeRef as a base opaque Core Foundation type, similar to NSObject in that all other core foundation objects are derived from this type, and often used similarly to 'id'. Note the "copy" in the function means that the returned object must be released when we are finished with it.

	CFStringRef KeychainPasswordForKeychainItem(CFStringRef serviceString, CFStringRef accountString, CFErrorRef *error)
	{
		OSStatus status = KeychainErrorBadArguments;
		CFStringRef resultString = NULL;
		
		if (0 < CFStringGetLength(serviceString) && 0 < CFStringGetLength(accountString))
	    {
			CFDataRef passwordData = NULL;
			CFMutableDictionaryRef keychainQueryDictionary = _QueryDictionaryForService(serviceString, accountString);
	        
	        if (keychainQueryDictionary)
	        {
	            CFDictionarySetValue(keychainQueryDictionary, kSecReturnData, kCFBooleanTrue);
	            CFDictionarySetValue(keychainQueryDictionary, kSecMatchLimit, kSecMatchLimitOne);
	            
	            //this will crash if user happens to lock phone by the time this is called
	            status = SecItemCopyMatching(keychainQueryDictionary, (CFTypeRef *)&passwordData);
	        }
	        
			if (status == noErr && 0 < CFDataGetLength(passwordData))
	        {
				resultString = CFStringCreateWithBytes(kCFAllocatorDefault, CFDataGetBytePtr(passwordData), CFDataGetLength(passwordData), kCFStringEncodingUTF8, TRUE);
	            CFAutorelease(resultString);
			}
			
			if (passwordData != NULL)
	        {
				CFRelease(passwordData);
			}
		}
		
		if (status != noErr && error != NULL)
	    {
			*error = CFErrorCreate(kCFAllocatorDefault, kKeychainServiceErrorDomain, status, NULL);
		}
		
		return resultString;
	}
	
Note a few things about this code. We are also setting the kSecReturnData to kCFBooleanTrue. kSecReturnData means the actual data of the item will be returned. Other options would be to return the attributes (kSecReturnAttributes) of the item, for example. These keys take CFBooleanRefs, which hold the constants kCFBooleanTrue or kCFBooleanFalse. These constants are ways of packaging or wrapping primitive Boolean types (bool or Boolean for example) into container or collection objects. We are also setting kSecMatchLimit to kSecMatchLimitOne to return only the first item found in the keychain as opposed to an unlimited number of results.

That is it for storing strings into the keychain. To use these helper functions, you can download the full XCode project [here](https://github.com/CollinBStuart/KeychainServices)

### Summary

In summary, the keychain is a great place to store information without having to worry about implementing protection yourself. However, keep in mind that there are weaknesses in storing information in the keychain. Please see [iOS Keychain Weakness FAQ](http://sit.sit.fraunhofer.de/studies/en/sc-iphone-passwords-faq.pdf). There are many attacks against the keychain and also professional forensics tools to uncover keychain data. The keychain can be decrypted and viewed as [this page](https://code.google.com/p/iphone-dataprotection/) explains. 

Even more specifically, the file system (including the keychain) on a device is encrypted using an AES256 key. That key is itself encrypted using the user's passcode, or with the fingerprint technology which deduces to the same strength as a 5 or 6 alphanumeric password. This means that the strength of the encryption may lie at the mercy of the user's passcode strength! Four digit numerical pass-codes can be brute forced in minutes. While in addition the OS prevents access to keychain information for each application based off of entitlements, this can easily be fooled, as in the example of [Dumping Keychain Data](http://resources.infosecinstitute.com/ios-application-security-part-12-dumping-keychain-data/). Therefore it is much better to store hashes in the keychain, then the hashes themselves need to additionally be brute forced or checked against a rainbow table. If the information is extremely sensitive, it may be better to implement encryption using PKI or another approach, depending on the context.
