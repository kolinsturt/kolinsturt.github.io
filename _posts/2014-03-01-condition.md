---
layout: post
category : lessons
tagline: "C++ Atomic Programming"
description: Condition variables in C++ 11 by Kolin Stürt
tags : [C++ 11, "std::condition_variable", Condition Variable, Multithreading, concurrent Programming]
---
{% include JB/setup %}

## Condition Variables

A critical background task may need to wait until another process is completed in order to continue, such as waiting for user authentication or until a virus scan is complete. The simplest way to do this would be to run a loop until the condition is met and then continue. Once the condition is met, it proceeds to the next part of the code. The problem with this is that the thread is taking up CPU cycles performing a loop while the CPU could be giving that time to threads that have actual work to do. This concept is called busy waiting. If an attacker is able to start many login requests but never complete them, it could fill up the CPU time with busy loops, leading to a denile-of-service attack. Rate limiting connections and setting timeouts may be solutions, but to prevent the thread from busy waiting, C++ 11's thread library has a *condition_variable*. std::condition_variable will block a thread until a notification is received in such a way that CPU time is preserved. To do this, we use the wait member function. For example:

	cv.wait(lk); //lk is the lock

Undefined behavior can also lead to security vulnerabilities. This is true when there are potential code paths in a security application that are overlooked, such as getting to a state where the user should only be able to by logging in. For condition variables, the most common cause of undefined behaviour are spurious wakeups. With many thread APIs, POSIX and Windows included, a thread can be woken up periodically even through no thread signalled the condition variable. Because of this complication, it is necessary to check to make sure the variable you are testing against is actually correct. Let's look at an example. Say we are testing a bool called ready. Passing in a [Lambda function](http://en.cppreference.com/w/cpp/language/lambda), we can do this easily:

	cv.wait(lk, []{return ready;});

Which is equivalent to:

	while(!ready)
	{
	        cv.wait(lk);
	}

As attackers will exploit race conditions, to analyze a shared condition variable across multiple threads, we will need to acquire a lock. The wait function will actually take care of the task of unlocking the mutex and suspending the thread until a notification is received to wake it up, at which point the lock will be reacquired.

	#include <iostream>
	#include <thread>
	
	std::mutex m;
	std::condition_variable cv;
	bool ready = false;
	void _WaitingThread()
	{
	    std::unique_lock<std::mutex> lk(m);
	    std::cout << "Waiting..." << std::endl;
	    cv.wait(lk, []{return ready;});
	    std::cout << "We can now do more work" << std::endl;
	}
	
	void _SignalingThread()
	{
	    std::lock_guard<std::mutex> lk(m);
	    
	    //do some work...
	    std::chrono::milliseconds sleepDuration(3000);
	    std::this_thread::sleep_for(sleepDuration);
	    
	    ready = true; //we are now ready
	    std::cout << "Notifying..." << std::endl;
	    cv.notify_one(); //notify sleeping thread we are ready to process
	}
	
	void StartThreads()
	{
	    std::thread t1(_WaitingThread);
	    std::thread t2(_SignalingThread);
	    t1.detach();
	    t2.detach();
	}

In this code, you used the wait method that safely checks against the ready flag. The notify_one function will notify one thread to wake up. A condition variable can block multiple threads at the same time, and to notify all waiting threads we can use the notify_all() function.

You will notice here we used something called chrono (instead of say, a usleep() function). C++ 11 introduces the chrono library which can be quite helpful in many scenarios. It allows access to the system-wide real time wall clock as well as steady monotonic and high-resolution clocks. 

Another way a process can be tied up leading to denile-of-service is in the event of a lost wakeup. This could happen if the notification is sent before the waiting started, or in the event an attacker can prevent the event such as a login to ever complete. We can wait until a specific time or until a predefined timeout occurs. wait_for() will unblock the thread when a specific amount of time has passed, whereas wait_until() can be used to set a specific date. In either case, if the notification is sent before the timeout, the thread will unblock just as it does with the wait() function. A steady clock is used for the duration. There are many units of time we can use, such as nanoseconds, microseconds, milliseconds, seconds, minutes and hours. Check out the [std::chrono library](http://en.cppreference.com/w/cpp/chrono) for all the options you can use.

As with other parts of this higher-level thread library, a native_handle() function will return you the underlying handle. For example, on iOS, Mac OS or any POSIX system, this will be a pthread_cond_t * variable. On Windows, it is PCONDITION_VARIABLE.

Denile-of-service attacks can happen the other way, for example, a login already succeeded yet the attacker keeps spawning busy loops that will never complete because a logged-in notification will never be sent again. You can have a function only ever called once – even across multiple threads - even if two threads call it at the same time. This is also good for initialization during a singleton class:

	std::once_flag flag;
	void SomeSingletonClass::sharedSingleton()
	{
	    std::cout << "This is called each and every time" << std::endl;
	
	    std::call_once(flag, [](){std::cout << "Init stuff. This is called only once" << std::endl;});
	}

The call_once() function utilizes the [std::once_flag](http://en.cppreference.com/w/cpp/thread/once_flag) which coordinates to make sure that a function only gets called once and runs to completion. Any [C++ Callable object](http://en.cppreference.com/w/cpp/concept/Callable) can be used as the second parameter. Here we decide to pass in a [Lambda function](http://en.cppreference.com/w/cpp/language/lambda). You can use this to make sure the state of an object doesn't get reinitialized.

For more information, check out [condition_variable](http://en.cppreference.com/w/cpp/thread/condition_variable)
