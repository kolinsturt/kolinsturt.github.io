---
layout: post
category : lessons
tagline: "iOS Data Protection and Privacy"
title: Overlooked Areas of Data Privacy: Keyboard and Screenshot Caches
description: Data protection and privacy in iOS
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
---
{% include JB/setup %}

## Overlooked Areas of Data Privacy

The subject of privacy and protecting the user’s data is a hot topic these days. Personal information that gets compromised is a big deal. I covered the subject of data privacy in my [iOS Data Privacy](http://code.tutsplus.com/articles/securing-ios-data-at-rest-protecting-the-users-data--cms-28527) and [Data Privacy for Android](https://www.raywenderlich.com/6901838-data-privacy-for-android) tutorials. While data privacy is covered in general in many books and articles, there's a few areas on iOS where you can leak the user’s data without realizing.

### Screenshots

iOS takes a screenshot of the app. That's so it can animate the app coming and going from background and show a preview in the task switcher. This image gets stored onto the phone. Stored data is susceptible to offline data acquisition. Make sure you hide any user sensitive data so that it does not get captured by the screenshot. To do this, set up a notification when the app is going into background. Then hide the UI with the `setHidden:YES` method for elements you want to exclude. Un-hide them when coming to the foreground. Start by setting up the notifications:

    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(gotoBackground) name:UIApplicationDidEnterBackgroundNotification object:nil];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(comebackfromBackground) name:UIApplicationWillEnterForegroundNotification object:nil];
    
Remove the notifications when the view disappears:
    
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationDidEnterBackgroundNotification object:nil];
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationWillEnterForegroundNotification object:nil];
    
If you're in Core Foundation:
    
    CFNotificationSuspensionBehavior suspendBehavior = CFNotificationSuspensionBehaviorDeliverImmediately;
    CFNotificationCenterRef center = CFNotificationCenterGetLocalCenter();
    CFNotificationCenterAddObserver(center, (__bridge const void *)(self), BackgroundNotificationCallback, (CFStringRef)UIApplicationDidEnterBackgroundNotification, NULL, suspendBehavior);
    
Cleanup:
    
    CFNotificationCenterRemoveObserver(CFNotificationCenterGetLocalCenter(), (__bridge const void *)(self), (CFStringRef)UIApplicationDidEnterBackgroundNotification, NULL);

### Keyboard Cache

Text fields by default have auto-correct. iOS stores text that the user types, such as learned words, in a cache. That means it's possible to retrieve some of the text that the user has entered in your app. To prevent this, disable the auto-correct option: 
	
    [myTextField setAutocorrectionType:UITextAutocorrectionTypeNo];
	
Sensitive information entered into text fields should use secure text entry to hide the visiable characters typed:

    UITextField *textField;
    [textField setSecureTextEntry:YES];

This setting also disabled auto-correct.
 
### File Protection

You can enable data protection across your app with one setting by enabling data protection for the app id in the Apple provisioning portal. Then in Xcode’s Project Settings, select the **Capabilities** tab and turn on **Data Protection**.

But if you enable data protection in newer projects but import old code, you may overlook disabling that protection. When you set up a Core Data Model, you can explicitly customize file protection with the `NSFileProtectionKey` constant. Make sure you don't import code that downgrades the security unless it's intentional:

    NSURL *storeURL = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"Model.sqlite"];
    
    // Make sure the database is encrypted when the device is locked
    if (![[NSFileManager defaultManager] setAttributes:@{NSFileProtectionKey : NSFileProtectionComplete} ofItemAtPath:[storeURL path] error:NULL])
    {
        ;
    }
    
You can do the same creating files and directories:
 
	BOOL ok = [[NSFileManager defaultManager] createFileAtPath:someFile contents:nil attributes:@{NSFileProtectionKey : NSFileProtectionComplete}];

    
    if (![[NSFileManager defaultManager] fileExistsAtPath:somePath])
    {
        [[NSFileManager defaultManager] createDirectoryAtPath:somePath withIntermediateDirectories:YES attributes:@{NSFileProtectionKey : NSFileProtectionComplete} error:nil];
    }

`NSData` has a method that writes it’s data to a file. You can also customize data protection at this point:

	NSData *data = ...;

	[data writeToFile:path options:(NSDataWritingAtomic | NSDataWritingFileProtectionComplete) error:&error];

You can enable data protection across your app with one setting by enabling data protection for the app id in the Apple provisioning portal website. Then in XCode's Project Settings, select the "Capabilities" tab and turn on "Data Protection"

Keep in mind that this type of data protection requires that the user has a passcode set on the device. If you would like to have control over when you apply encryption, see the [Securing iOS Data at Rest: Encryption](http://code.tutsplus.com/tutorials/securing-ios-data-at-rest-encryption--cms-28786) and [Encryption Tutorial or Android](https://www.raywenderlich.com/778533-encryption-tutorial-for-android-getting-started).

For information about app data privacy in general, check out the iOS [Protecting The User's Data](http://code.tutsplus.com/articles/securing-ios-data-at-rest-protecting-the-users-data--cms-28527) and [Data Privacy for Android](https://www.raywenderlich.com/6901838-data-privacy-for-android) tutorials.
