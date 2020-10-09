---
layout: post
category : lessons
tagline: "C++ condition variables"
title: Secure Coding with Concurrency in C++ - Condition Variables
description: Safe coding with std::condition_variable in C++11
author: Kolin Stürt
tags : [C++ 11, "std::condition_variable", Condition Variable, Multithreading, concurrent Programming]
---
{% include JB/setup %}

## Condition Variables

In the previous [article](https://kolinsturt.github.io/lessons/2014/02/01/mutex), you learned how to synchronize data. In this article you'll use those techniques to coordinate critical background work.

A critical background task may need to wait until another process is completed in order to continue, such as waiting for authentication to complete. This is often implemented by a run-loop that has a callback when the system meets a condition. One the system meets the condition, it proceeds to the next part of the code. The problem with this is that a thread is taking up CPU cycles performing a loop, called busy waiting. The CPU could be giving that time to threads that have actual work to do. 

#### Denial Of Service

If an attacker is able to start many authentication requests but never complete them, it could fill up the CPU time with busy loops, leading to a denial of service attack. Rate limiting the requests and setting timeouts are part of the solution, but to prevent the thread from busy waiting, C++ 11’s thread library has a `condition_variable`. 

`std::condition_variable` will block a thread until a notification is received in such a way that it preserves CPU time. You'll use the wait member function to accomplish this:

	cv.wait(lk); //lk is the lock

Undefined behavior can also lead to security vulnerabilities - when there are potential code paths in a security application that QA overlooked. An example is reaching a logged-in area when you're not logged-in. For condition variables, the most common cause of undefined behavior are spurious wakeups. 

#### Spurious Wakeups 

With many thread APIs, POSIX and Windows included, the platform may wake the thread up periodically even through no thread signaled the condition variable. Because of this, it's necessary to make sure the variable you are testing against is actually correct. You are testing a `bool` called `ready`. Passing in a [Lambda function](http://en.cppreference.com/w/cpp/language/lambda), you can do this easily:

	cv.wait(lk, []{return ready;});

Which is similar to:

	while(!ready)
	{
	        cv.wait(lk);
	}

Attackers will exploit race conditions. To analyze a shared condition variable across threads, you'll need to aquare a lock. The wait function will take care of unlocking the mutex and suspending the thread until until it receives a notification to wake up. The function will reacquire the lock when woken up.

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

In this code, you used the `wait` method that safely checks against the `ready` flag. The `notify_one` function will notify one thread to wake up. A condition variable can block multiple threads at the same time, and to notify all waiting threads we can use the `notify_all` function.

You used something `std::chrono` instead of a `usleep()` function. C++ 11 introduces the chrono library which is helpful in many scenarios. It allows access to the system-wide real-time wall clock as well as steady monotonic and high-resolution clocks.

#### Lost Wakeups

An attacker can tie a process up, causing a denial of service, by exploiting a lost wakeup. This could happen if the system sends the notification before the waiting started, or in the event an attacker can prevent the event such as authentication never completing. You can wait until a specific time or until a predefined timeout occurs. `wait_for` will unblock the thread when a specific amount of time has passed, whereas you can use `wait_until` to set a specific date. In either case, if the platform sends the notification before the timeout, the thread will unblock as it does with the `wait` function. 

The functions use a steady clock for the duration. There are many units of time you can use: nanoseconds, microseconds, milliseconds, seconds, minutes and hours. Check out the [std::chrono library](http://en.cppreference.com/w/cpp/chrono) for all the options.

Consistent with other higher-level APIs, a `native_handle` function will return you the underlying handle. For example, on iOS, Mac OS or any POSIX system, it returns a `pthread_cond_t *` variable. On Windows, it's `PCONDITION_VARIABLE`.

#### Call-Once Functions

Denial of service attacks can happen the other way. A authentication already succeeded but the attacker keeps spawning busy loops that will never complete because you're already authenticated. You can have a function only ever called once, even across multiple threads, even if two threads call it at the same time. This is good practice for initialization of a singleton class:

	std::once_flag flag;
	void SomeSingletonClass::sharedSingleton()
	{
	    std::cout << "This is called each and every time" << std::endl;
	
	    std::call_once(flag, [](){std::cout << "Init stuff. This is called only once" << std::endl;});
	}


The `call_once` function utilizes the [std::once_flag](http://en.cppreference.com/w/cpp/thread/once_flag) which makes sure a function only gets called once and runs to completion. You can use any [C++ Callable object](http://en.cppreference.com/w/cpp/concept/Callable) as the second parameter. In the example you've passed in a [Lambda function](http://en.cppreference.com/w/cpp/language/lambda). You used it to make sure the state of the object doesn’t get reinitialized.

You've learned to coordinate critical work. Directing work often requires flags that determine the state of a job or data. Check out the [Atomic Variables article](https://kolinsturt.github.io/lessons/2013/03/01/atomic) to learn how to safely use flags in a multi-threaded context. If you'd like to know more about `condition_variable`, check out the [condition_variable reference](http://en.cppreference.com/w/cpp/thread/condition_variable).
