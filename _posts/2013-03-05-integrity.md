---
layout: post
category : lessons
tagline: "Application Integrity"
title: App integrity on iOS
description: iOS application integrity
author: Kolin Stürt
tags : [iOS, Foundation, Core Foundation, Secure Programming]
---
{% include JB/setup %}

## Application Integrity

Depending on the sensitivity of the application, it may be in your best interest to make sure it's not operating in a tampered environment. For instance, a malicious user might have attached a debugger or jailbroken the device. Often spammers or reverse engineers tamper with the environment. Once it appears that the integrity has been compromised, you can choose appropriate actions. Some examples are limiting functionality, wiping data or phoning home.

### Detecting Jailbroken Devices

To check for jailbroken devices, test that a private area in storage is writable, such as **/private/jailbreak.txt**:

    NSError *error = nil;
    [@"Test" writeToFile:pathString atomically:YES encoding:NSUTF8StringEncoding error:&error];
    if ( ! error)
    {
        //jailbroken
        [[NSFileManager defaultManager] removeItemAtPath:pathString error:nil];
    }

Check for files often installed by a jailbreak:

    "/bin/bash"
    "/usr/sbin/sshd"
    "/etc/apt"
    "/private/var/lib/apt/"
    "/var/cache/apt"
    "/var/lib/apt"
    "/usr/bin/sshd"

You can also look for the most common jailbreak apps:

	static const char* jailbreakApps[] = 
     {
         "/Applications/Sileo.app"
         "/Applications/Cydia.app",
	 "/private/var/lib/cydia",
	 "/var/lib/cydia",
         "/Applications/limera1n.app",
         "/Applications/greenpois0n.app",
         "/Applications/blackra1n.app",
         "/Applications/blacksn0w.app",
         "/Applications/redsn0w.app",
         NULL,
     };
     
You can use `NSFileManager`’s `fileExistsAtPath` or `stat()`:

    Boolean isJB = FALSE;
    struct stat theStat;
    int result = (stat("/Applications/Cydia.app", &theStat) == 0);
    if (result)
    {
        isJB = TRUE;
    }
    
Most jailbreak patches install the Mobile Substrate dynamic library. This is a common library that allows programmers to make runtime patches to system functions. This can be dangerous if your application is relying on system functions. You can look for this library to try to determine if a device is jailbroken:

    result = (stat("/Library/MobileSubstrate/MobileSubstrate.dylib", &theStat) == 0);
    if (result)
    {
        isJB = TRUE;
    }
    
Most redsn0w jailbreaks move the application folder from the small read-only partition to the large one on the device and symbolic link it. You can check for this using the link version of the stat function, `lstat()`:

	if (lstat("/Applications", &theStat) != 0)
    {
        if (theStat.st_mode & S_IFLNK) //mode of the file is a symbolic link
        {
            isJB = TRUE;
        }
    }
    
[Johnathan Zdziarski](http://www.zdziarski.com/blog/?cat=8) researched that the size of the fstab on iOS (so far) has always been 80 bytes but noticed that on jailbreak devices, fstab is 65 bytes. That's because the jailbreak patches it for read-write access:

    stat("/etc/fstab", &theStat);
    off_t fstabSize = theStat.st_size;
    if (fstabSize < 80)
    {
        ; //possibly jailbroken
    }
    
He also mentions that iOS disables many functions like `fork()` for the app sandbox environment. Therefor if `fork()` succeeds, the phone may be jailbroken. You can use this check for adhoc builds. You don’t want to `fork()` a process without cleanup by calling `exit(0)`. Explicitly exiting an app is against Apple’s review guidelines and you'll get rejected for calling exit function.:

	#if !(TARGET_IPHONE_SIMULATOR)
	    pid_t theID = fork();
	    if (theID >= 0) //if fork fails, it returns a negative number
	    {
	        isJB = TRUE;
	        //since this succeeds, we should exit(0) the forked process, but Apple will reject us for that code so it is not safe to use
	    }
	#endif

Make sure that you exclude your tests from debug and simulator builds to prevent false positives.

### Detecting Debuggers

There's a few things we can try to do to detect if [a debugger is attached](https://developer.apple.com/library/mac/qa/qa1361/_index.html). The kernel sets a `P_TRACED` flag when you attach a debugger so you can check for this:

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

Apple created a `PT_DENY_ATTACH` flag for Mac OS that you could use to deny tracing and debugging. It doesn't exist on iOS but it’s value is 31 so you can define it in your code. 

Use the ptrace system call to pass in a request integer to the first parameter - `PT_DENY_ATTACH`. You'll want to have your own flag to disable it for debug environments. Add the function before the main run loop to prevent the application from launching if it detects a debugger:

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

Depending on your app, it may be completely fine for users to use your product on a jailbroken phone. Integrity checking in recent years helps prevent spammers from botting an app.

To learn more about protecting your environment from spammers, check out the [Digital Signatures With Swift](http://code.tutsplus.com/tutorials/creating-digital-signatures-with-swift--cms-29287), [Securing Network Data Tutorial for Android](https://www.raywenderlich.com/10056112-securing-network-data-tutorial-for-android) and [SafetyNet API documentation](https://developer.android.com/training/safetynet/attestation).
