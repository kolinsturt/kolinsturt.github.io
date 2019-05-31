---
layout: post
category : lessons
tagline: "NSSecureCoding"
description: Implementing NSSecureCoding - Kolin Stürt.
tags : [NSSecureCoding, decodeObjectOfClass, NSKeyedArchiver, NSKeyedUnarchiver, supportsSecureCoding, requiresSecureCoding, iOS]
---
{% include JB/setup %}

## NSSecureCoding

There exists an attack vector with NSCoder in which it can blindly deserialize an object from disk. An attacker just has to modify the underlying plist of the stored data so that a malicious object is constructed at runtime. Even with performing validation with the isKindOfClass: check, the object needs to already exist at this point. A malicious object could have deliberate errors included to cause an overflow upon construction which the attacker could then use to escalate an attack.

NSSecureCoding aims at closing this hole. It at least provides diligent checking and is a fair enough fix, but it doesn't protect against other runtime hacks such as overriding isKindOfClass which could render the internal checks useless. In any case it can help make your code more robust and help make sure your classes are designed by contract.

Inside your custom class, conform to the secure coding protocol in your header file

	@interface MyClass : NSObject <NSSecureCoding>

Then override the protocol method in your implementation file to enforce secure coding

	+ (BOOL)supportsSecureCoding
	{
	    return YES;
	}

Once you adopt the protocol, you must return YES from this class method, and if your custom objects use initWithCoder:, the decodeObjectForKey: methods will throw an exception. You must now use decodeObjectOfClass: forKey: instead. In this way you can establish the contract for the object types ahead of time

	-(id) initWithCoder:(NSCoder *)decoder
	{
	        [self setMyString:[decoder decodeObjectOfClass:[NSString class] forKey:(NSString *)glstrKeyMyString]];
		[self setMyNumber:[decoder decodeObjectOfClass:[NSNumber class] forKey:(NSString *)glnumKeyMyNum]];
	        //...
	}

For optimization purposes, unarchivers have direct support for unpacking NSStrings, NSNumbers and NSData as they are not encoded but stored directly in a binary property list. This means you could swap an [NSNumber class] with an [NSString class] for decodeObjectOfClass: and it would still work.

Additionally, if you encode the same object multiple times, the decode check only happens once per instantiated object. Subsequent decode calls to an object that is already unpacked will pass even if you swap the object type.

When archiving and unarchiving your custom object to and from the file system, you will need to do this in a specific way in order to enable secure coding. In the past you may have used [NSKeyedUnarchiver unarchiveObjectWithFile:fileString]. This method has secure coding turned off by default. We will explicitly turn it on by passing yes to NSKeyedUnarchiver's setRequiresSecureCoding method.

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

That is all you need to do to implement secure coding. One addition you can make is to also perform the archiving with secure coding turned on. All this does is prevent you from accidentally archiving an object that does not adhere to the secure coding protocol. You may have previously used NSKeyedArchiver's archiveRootObject. Here is an example that would go inside your custom object that shows how to accomplish the same archiving of your object but with secure coding turned on

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

Note that the class methods archiveRootObject and  unarchiveObjectWithFile by default internally use the NSKeyedArchiveRootObjectKey key. Therefore when we alloc init these objects ourselves, we can still use the same functionality by supplying the  NSKeyedArchiveRootObjectKey key and it will be backward compatible with unarchiving previously stored objects that used the  archiveRootObject class methods.

That's it for NSSecureCoding. Keep in mind it's always good to implement your own data validation checks upon unpacking any archive or receiving any arbitrary input in general.
