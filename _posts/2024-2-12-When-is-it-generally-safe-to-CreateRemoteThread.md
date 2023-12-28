---
layout: post
title: When is it generally safe to CreateRemoteThread?
image: /images/When-is-it-generally-safe-to-CreateRemoteThread/A-chart-showing-the-flow-of-an-early-injection-with-CRT.png
---

In this short blog post I want to share interesting observations regarding remote thread creation. Some of the information presented here was already mentioned in previous blog posts, but I thought having a dedicated post about it can serve as a useful reference.

So, when is it safe to create a remote thread in an arbitrary process? If asked a couple of years ago, I'd probably say that it doesn't matter, and that creating a remote thread in any process at any time is generally fine, unless the process does something non-standard such as having anti-debugging or anti-tampering mechanisms. But gaining some experience with [global injection in Windows]({{ site.baseurl }}/Implementing-Global-Injection-and-Hooking-in-Windows/) led me to discover that it's not that simple.

## Process initialization and csrss.exe

Creating a remote thread in a console process which isn't fully initialized may cause the process to fail to start.

Normally, a new process is created with the `CreateProcess` function (or one of its variants). A good overview of the process creation flow can be found in the [Genesis - The Birth Of A Windows Process (Part 2)](https://fourcore.io/blogs/how-a-windows-process-is-created-part-2) blog post by Hardik Manocha. Here is a partial list of steps that are performed during process creation:

* Parameters and flags are converted and validated.
* `NtCreateUserProcess` is called to preform steps in kernel mode.
  * The target exe file is opened and a section object is created.
  * The initial thread and its stack and context are created.
* Windows subsystem-specific initialization is performed.
  * A message is sent to the Client/Server Runtime Subsystem (`csrss`) to notify it about the new process.
  * `csrss` performs its own initialization, such as allocating structures for the new process.
* The initial thread is resumed.
* Process initialization is performed in the context of the new process.

All user mode threads begin their execution in the `LdrInitializeThunk` function. The first thread that a process runs performs process initialization tasks before the execution is transferred to the user-supplied thread entry point. One of the process initialization tasks is creating the console window in case the process is a console process. For more details about the `LdrInitializeThunk` function, check out [this blog post](http://www.nynaeve.net/?p=205) by Ken Johnson.

If a remote thread (marked with red in the image below) is created and starts executing before `csrss` gets notified about the new process, then as the first running thread, it performs the process initialization tasks. Only then, `csrss` gets notified about the new process, but it doesn’t expect the process to have an initialized console, and returns an error. As a result, the process is terminated and the process creation fails.

![A chart showing the flow of an early remote thread execution]({{ site.baseurl }}/images/Implementing-Global-Injection-and-Hooking-in-Windows/A-chart-showing-the-flow-of-a-too-early-injection.png){:width="633px"}

## CRT initialization and TLS

OK, so creating a remote thread so early, even before `CreateProcess` returns, may cause problems. What if we manage to make sure `CreateProcess` returns successfully? As it turns out, sometimes that's not enough.

At some point, I [got reports about crashes]({{ site.baseurl }}/A-guest-in-another-process-a-story-of-a-remote-thread-crash/) caused by my global injection implementation. At first, I thought that such crashes are rare and limited to programs which make unusual use of TLS (Thread Local Storage) callbacks, but eventually I realized that this kind of problem may also happen with a regular program which does nothing special. In fact, it may even happen with a Hello World program, depending on the CRT (C Runtime) it's linked with. Below is a specific example.

A program that is statically linked with Microsoft's UCRT (Universal C Runtime) initializes the CRT before the execution is transferred to the user-supplied entry point (the `main`/`WinMain` function). A global variable is initialized with the process heap during initialization. Afterwards, each newly created thread triggers a TLS callback which uses the global variable during memory allocation.

```c
// Initializes the heap.  This function must be called during CRT startup, and
// must be called before any user code that might use the heap is executed.
extern "C" bool __cdecl __acrt_initialize_heap()
{
    __acrt_heap = GetProcessHeap();
    // ...
}
```

```c
// This function implements the logic of malloc().  It is called directly by the
// malloc() function in the Release CRT and is called by the debug heap in the
// Debug CRT.
// ...
extern "C" __declspec(noinline) _CRTRESTRICT void* __cdecl _malloc_base(size_t const size)
{
    // ...
    for (;;)
    {
        void* const block = HeapAlloc(__acrt_heap, 0, actual_size);
        // ...
    }
}
```

As you probably noticed, the problem occurs if a new thread is created before the `__acrt_heap` variable is initialized. In this case, `HeapAlloc` is called with an invalid heap, and the program crashes.

![A chart showing the flow of an early thread execution]({{ site.baseurl }}/images/When-is-it-generally-safe-to-CreateRemoteThread/A-chart-showing-the-flow-of-an-early-injection-with-CRT.png){:width="433px"}

This example demonstrates that even in the standard case of a simple C/C++ program, creating a remote thread may cause problems.

## Conclusions

Creating a remote thread in an arbitrary process is never 100% safe (I showed a contrived example [here]({{ site.baseurl }}/A-guest-in-another-process-a-story-of-a-remote-thread-crash/#thread-initialization-and-tls-callbacks)), but this blog post shows that it may not be safe even for simple programs that do nothing unusual. In the general case, one at least has to make sure that CRT initialization (or equivalent) is completed before creating a remote thread. You might want to keep this in mind the next time you use `CreateRemoteThread`.

## Workarounds

At this point, you might wonder how to avoid these pitfalls and make remote thread creation as safe as possible. There's no universal solution, but here are things that I explored, that might or might not be helpful for you depending on your needs:
* [Making sure that the target process started executing]({{ site.baseurl }}/Implementing-Global-Injection-and-Hooking-in-Windows/#too-early-injections-that-break-stuff).
* [Creating a thread that doesn't trigger TLS callbacks]({{ site.baseurl }}/A-guest-in-another-process-a-story-of-a-remote-thread-crash/#looking-for-a-fix).
