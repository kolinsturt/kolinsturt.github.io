---
layout: post
category : lessons
tagline: "Data Protection"
title: iOS data protection
description: Data protection in iOS
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
---
{% include JB/setup %}

## Data Protection

The subject of privacy and protecting the user's data is a hot topic these days. Personal information that gets compromised is a big deal. There are some areas of iOS where user's data may be captured without us realizing. Some examples are animation screenshots and keyboard caches.

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
