---
layout: post
category : lessons
tagline: "Keychain Services in Core Foundation"
title: Keychain Services in C
description: Keychain Services C API on iOS
author: Kolin Stürt
tags : [Keychain Services, SecItem, ios, Core Foundation, SecItemCopyMatching]
---
{% include JB/setup %}

## Keychain Services in C

In this article, you'll learn how to use the keychain services C API.

### Storing passwords

There are many Foundation wrappers for the keychain services such as [SSKeychain](https://github.com/soffes/sskeychain). If you're working in a C/Core Foundation environment, you can create your own convenience functions.
The keychain as a typical database under the hood that runs queries on a table. It's an SQLite Database stored on the iPhone at /private/var/Keychains/keychain-2.db and /users/USERNAME/Library/Keychains/ on OS X. 

The functions of the keychain services API take a `CFDictionary` which specifies the keychain identifiers and attributes of each function. For example, if you're storing a password, you set the keychain class, `kSecClass`, to `kSecClassGenericPassword`. 

Each entry in the keychain has a service name. The “service name” is an identifier; a “key” for whatever “value” you want to store in the keychain. To make sure that your keychain items get stored for your app only and to prevent duplicate entries, you need to specify an account name. The “account name” would be an identifier for your app. You could use a reverse DNS name to avoid conflicts such as **com.testcompany.keychaintest**.

Because each function takes a dictionary of parameters to make a query, it makes sense to avoid duplicate code. Make a query function that returns a CFDictionary now:

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

You adding an extra layer of security to the query dictionary so that keychain items are only accessible when the user unlocks their device. The `kSecAttrAccessible` key defines this behavior, where you passed `kSecAttrAccessibleWhenUnlocked`.

It's discouraged to store user's private information such as a password or phone number. If attackers compromise the keychain, they'll have access to the data. Instead, store the [hashed value](https://kolinsturt.github.io/lessons/2013/05/01/hashing_algorithms_in_core_foundation) and only compare the hashes for access control.

This brings up an important point about relying on a system. When everyone relies on one security framework and it gets compromised, all the software that rely on it are also compromised. That's one reason to use a mixture of audited frameworks such as [OpenSSL](https://kolinsturt.github.io/lessons/2011/11/15/aes_encryption_using_openssl);

To get started, create some globals for error handling. Taking an idea from the SSKeychain library, Sam Soffes added `BadArguments` and `NoPassword` to the enum of errors from **SecBase.h**. Define them now:

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


Now you'll store a string by using the `SecItemAdd()` function. It takes a `CFDataRef` object for versatility. Convert your string into data and pass it in as a parameter to the query dictionary:

    CFDataRef passwordData = CFStringCreateExternalRepresentation(kCFAllocatorDefault, passwordString, kCFStringEncodingUTF8, 0);
    CFDictionarySetValue(keychainQueryDictionary, kSecValueData, passwordData);
    SecItemAdd(keychainQueryDictionary, NULL);
    
Adding error checking, it looks like this:

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
	
#### Deleting Items
	
To prevent duplicate inserts you first delete the previous entry if there is one. To do that, you use the `SecItemDelete()` function:

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

Now all that's left is to retrieve an entry from the keychain. You this using the `SecItemCopyMatching()` function. It will return a `CFTypeRef` base object. In your case this will be a `CFStringRef` instance.

`CFTypeRef` is the base opaque Core Foundation type the same as `NSObject`. All other Core Foundation objects derive from it. The “copy” in the function means that you must `CFRelease` the returned object when you no longer need it:

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
	
Here's a few things about this code:

- You are setting `kSecReturnData` to `kCFBooleanTrue`. `kSecReturnData` means to return the actual data of the item.
- Another option is `kSecReturnAttributes` to return the properties of the item.
- These keys take `CFBooleanRef``, which hold the constants `kCFBooleanTrue` or `kCFBooleanFalse`. 
- These constants are ways of packaging primitive `Boolean` types into container objects.
- You're also setting `kSecMatchLimit` to `kSecMatchLimitOne`. That returns only the first item in the keychain instead of a `CFArrayRef`.

To use these helper functions, download the full XCode project [here](https://github.com/CollinBStuart/KeychainServices)

### Summary

The keychain is a great place to store information without implementing the protection yourself. There have been [iOS Keychain Weakness FAQ](http://sit.sit.fraunhofer.de/studies/en/sc-iphone-passwords-faq.pdf). There are many professional forensics tools to [uncover keychain data](https://code.google.com/p/iphone-dataprotection/). 

Apple encrypts the file system on a device using an AES256 key. That key is encrypted using the user’s passcode, or fingerprint which deduces to the same strength as a 5 to 6 alphanumeric password. This means that the strength of the encryption is at the mercy of the user’s passcode strength. 

Four digit numerical pass-codes can be brute forced in minutes. The OS prevents access to keychain for each application based off of entitlements. [Dumping Keychain Data](http://resources.infosecinstitute.com/ios-application-security-part-12-dumping-keychain-data/) demonstrated a way to circumvent it.

If you can, store [hashes](https://kolinsturt.github.io/lessons/2013/05/01/hashing_algorithms_in_core_foundation) in the keychain. Then attackers would need to check them against a rainbow table (without a [salt](https://code.tutsplus.com/tutorials/securing-ios-data-at-rest-encryption--cms-28786)) or brute force them. If the information is extremely sensitive, it may be better to implement encryption using PKI or another approach. Check out the [Securing iOS Data at Rest](http://code.tutsplus.com/tutorials/securing-ios-data-at-rest-encryption--cms-28786) and [Encryption for Android tutorial](https://www.raywenderlich.com/778533-encryption-tutorial-for-android-getting-started) for more information.
