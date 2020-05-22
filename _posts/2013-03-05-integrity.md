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

Depending on the context or sensitivity of the application, it may be in your best interest to test the environment to make sure it has not been tampered with. Once it appears that the integrity has been compromised, the developer can choose to do a number of actions: limiting functionality, wiping data, phoning home, etc; an application's behavior can deliberately be changed if the application detects the integrity of the environment has been compromised. For example, a malicious user might have attached a debugger to the process to watch the contents of variables, or a mobile device may have been jailbroken. Lets start with the latter.

In order to check for jailbroken devices, we could look for the most common jailbreak applications that are installed 

	static const char* jailbreakApps[] = 
     {
         "/Applications/Cydia.app",
         "/Applications/limera1n.app",
         "/Applications/greenpois0n.app",
         "/Applications/blackra1n.app",
         "/Applications/blacksn0w.app",
         "/Applications/redsn0w.app",
         NULL,
     };
     
We can use NSFileManager's fileExistsAtPath, or we can search specifically for a file using the stat() command

	Boolean isJB = FALSE;
    struct stat theStat;
    int result = (stat("/Applications/Cydia.app", &theStat) == 0);
    if (result)
    {
        isJB = TRUE;
    }
    
Most jailbreak patches install the Mobile Substrate dynamic library. This is a common library that allows programmers to make runtime patches to system functions. This can be dangerous if your application is relying on system functions. We can look for this library to try to determine if a device is jailbroken

	result = (stat("/Library/MobileSubstrate/MobileSubstrate.dylib", &theStat) == 0);
    if (result)
    {
        isJB = TRUE;
    }
    
Most redsn0w jailbreaks move the application folder from the small readonly partition to the large one on the device and symbolic link it, so we can check for this using the link version of the stat function, lstat()

	if (lstat("/Applications", &theStat) != 0)
    {
        if (theStat.st_mode & S_IFLNK) //mode of the file is a symbolic link
        {
            isJB = TRUE;
        }
    }
    
[Johnathan Zdziarski](http://www.zdziarski.com/blog/?cat=8) had researched that the size of the fstab in iOS (so far) has always been 80 bytes but noticed that on jailbreak devices, fstab is 65 bytes because it has been patched for read-write access

	stat("/etc/fstab", &theStat);
    off_t fstabSize = theStat.st_size;
    if (fstabSize < 80)
    {
        ; //possibly jailbroken
    }
    
He also mentions that many functions like the fork() function to fork a process are turned off for the app sandbox environment of iOS, therefor is fork() succeeds, the phone may be jailbroken. This check may not be able to be used unless you are writing adhoc code that does not need to be approved through the App store as you don't want to fork() a process. If the fork process succeeds, you'd need to exit the process with exit(0), however explicitly exiting an app is against Apple's review guidelines and may get your app rejected. If your using this in a test environment, make sure that you exclude when the user is running an iPhone simulator as you will incorrectly be notified of a jailbreak when their isn't one

	#if !(TARGET_IPHONE_SIMULATOR)
	    pid_t theID = fork();
	    if (theID >= 0) //if fork fails, it returns a negative number
	    {
	        isJB = TRUE;
	        //since this succeeds, we should exit(0) the forked process, but Apple will reject us for that code so it is not safe to use
	    }
	#endif


Last but not least, there are a few things we can try to do to detect if [a debugger is attached](https://developer.apple.com/library/mac/qa/qa1361/_index.html). The kernel has a flag, P_TRACED, that is set when a debugger is attached and we can simple check for it


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

Another thing we can do is block the application from running if a debugger is attached in production environments. Apple created a flag, PT_DENY_ATTACH for Mac OS X that could be used for your application to request that it not be traced or debugged. On iOS it doesn't exist, but it's value is 31 so we can just define it in our code. We use the ptrace system call to pass in a request integer to the first parameter - PT_DENY_ATTACH. You will obviously want to have your own flag to turn this test off for debug environments and on for production environments. Putting this function before the main run loop is set up will prevent the application from launching if a debugger is attached. 

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

Depending on your app, it may be completely fine for users to use your product on a jailbroken phone. Judgment has to be made depending on the developer or company needs.
