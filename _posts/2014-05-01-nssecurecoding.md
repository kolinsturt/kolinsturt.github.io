---
layout: post
category : lessons
tagline: "NSSecureCoding on iOS"
title: NSSecureCoding findings
author: Kolin Stürt
description: Implementation findings of NSSecureCoding on iOS
tags : [NSSecureCoding, decodeObjectOfClass, NSKeyedArchiver, NSKeyedUnarchiver, supportsSecureCoding, requiresSecureCoding, iOS]
---
{% include JB/setup %}

## NSSecureCoding Findings

There's an attack vector in `NSCoder` where it blindly deserialize objects from disk. An attacker needs to modify the underlying plist of the stored data so that the runtime constructs a malicious object. Even if you perform validation with an `isKindOfClass:` check, the object needs to already exist at that point. A malicious object could have deliberate faults included to cause an overflow upon construction which the attacker could then use to escalate an attack.

`NSSecureCoding` aims at closing this hole. It at least provides diligent checking but it doesn’t protect against other runtime hacks such as overriding `isKindOfClass` which could render the internal checks useless. In any case it can help make your code more robust and follow a design-by-contract paradigm.

Inside your custom class, conform to the secure coding protocol in your header file:

	@interface MyClass : NSObject <NSSecureCoding>
	
Then override the protocol method in your implementation file to enforce secure coding:

	+ (BOOL)supportsSecureCoding
	{
	    return YES;
	}

Once you adopt the protocol, you must return `YES` from this class method. If your custom objects use `initWithCoder:`, the `decodeObjectForKey:` methods will throw an exception. You must now use `decodeObjectOfClass: forKey:` instead. This way you establish the contract for the object types ahead of time:

	-(id) initWithCoder:(NSCoder *)decoder
	{
	        [self setMyString:[decoder decodeObjectOfClass:[NSString class] forKey:(NSString *)glstrKeyMyString]];
		[self setMyNumber:[decoder decodeObjectOfClass:[NSNumber class] forKey:(NSString *)glnumKeyMyNum]];
	        //...
	}

For optimization, unarchivers have direct support for unpacking `NSString`, `NSNumber` and `NSData` as they are not encoded but stored in a binary property list. This means you could swap an `[NSNumber class]` with `[NSString class]` for `decodeObjectOfClass:` and it would still work.

Additionally, if you encode the same object multiple times, the decode check only happens once per instantiated object. Subsequent decode calls to an object that is already unpacked will pass even if you swap the object type.

When archiving and unarchiving your custom object to and from the file system, you will need to do this in a specific way in order to enable secure coding. In the past you may have used `[NSKeyedUnarchiver unarchiveObjectWithFile:fileString]`. This method has secure coding turned off by default. You'll need to enable it by passing `YES` to `NSKeyedUnarchiver`’s `setRequiresSecureCoding` method:

	+ (MyObject *)myObjectFromSavedData
	{
	    MyObject *myObject = nil;
	    NSString *fileString = [[self class] _pathToSavedData];
	    if ([[NSFileManager defaultManager] fileExistsAtPath:fileString])
	    {
	        // Set up NSKeyedUnarchiver to use secure coding
	        NSData *data = [NSData dataWithContentsOfFile:fileString];
	        if (data)
	        {
	            NSKeyedUnarchiver *unarchiver = [[NSKeyedUnarchiver alloc] initForReadingWithData:data];
	            if (unarchiver)
	            {
	                [unarchiver setRequiresSecureCoding:YES];
	                myObject = [unarchiver decodeObjectOfClass:[MyObject class] forKey:NSKeyedArchiveRootObjectKey];
	                [unarchiver finishDecoding];
	            }
	        }
	    }
	    return myObject;
	}

Now that you've enabled secure unarchiving, you can also perform the archiving with secure coding. All this does is prevent you from accidentally archiving an object that does not adhere to the secure coding protocol. You may have previously used `NSKeyedArchiver`’s `archiveRootObject`. Here's how to accomplish the same thing but with secure coding enabled:

	- (void)save
	{
	    NSString *documentsString = [[self class] _pathToSavedData];
	    NSError *error = nil;
	    NSMutableData *data = [NSMutableData data];
	    NSKeyedArchiver *archiver = [[NSKeyedArchiver alloc] initForWritingWithMutableData:data];
	    if (archiver)
	    {
	        [archiver setRequiresSecureCoding:YES];
	        [archiver encodeObject:self forKey:NSKeyedArchiveRootObjectKey];
	        [archiver finishEncoding];
	        BOOL success = [data writeToFile:documentsString options:(NSDataWritingAtomic | NSDataWritingFileProtectionComplete) error:&error];
	        if (success)
	        {
	            NSLog(@"Saving success");
	        }
	        else
	        {
	            NSLog(@"error saving object : %@", [error localizedDescription]);
	        }
	    }
	    else
	    {
	        NSLog(@"error setting up archive to save object");
	    }
	}
	
The class methods `archiveRootObject` and `unarchiveObjectWithFile` by default use `NSKeyedArchiveRootObjectKey` internally. When you instantiate your objects, you can supply `NSKeyedArchiveRootObjectKey` and it will be backward compatible. You can unarchive previous objects that used the `archiveRootObject` class methods.

You should still perform your own data validation checks when unarchiving arbitrary data. For more information about secure coding in general, see [Objective-C Input Validation](https://kolinsturt.github.io/lessons/2013/03/04/validation), [Secure Coding in Swift](http://code.tutsplus.com/tutorials/secure-coding-in-swift-4--cms-29835) and the [Null Safety Tutorial in Kotlin](https://www.raywenderlich.com/436090-null-safety-tutorial-in-kotlin-best-practices).
