---
layout: post
title: A guest in another process - a story of a remote thread crash
image: /images/A-guest-in-another-process-a-story-of-a-remote-thread-crash/Windhawk-advanced-options-process-list.png
---

In one of my previous blog posts, [Implementing Global Injection and Hooking in Windows]({{ site.baseurl }}/Implementing-Global-Injection-and-Hooking-in-Windows/), I wrote about my journey in implementing global DLL injection for [Windhawk](https://windhawk.net/), the customization marketplace for Windows programs. If you haven't read it yet, I invite you to read it, but the bottom line is that I ended up with an implementation that enumerates all processes and injects the DLL into each of them. To make sure the DLL is also loaded in newly created processes, the implementation intercepts new process creation and injects into each newly created process. A demo implementation can be found in the [global-inject-demo](https://github.com/m417z/global-inject-demo) repository.

Since the blog post was published, I encountered and fixed several interesting problems with the implementation, including:

* [A subtle bug in the wow64ext library](https://github.com/rwfpl/rewolf-wow64ext/pull/21) (try [the quiz](https://twitter.com/m417z/status/1522190986922897408) first).
* [An incompatibility with Firefox](https://twitter.com/m417z/status/1531161682600513536) due to its unique attempt at reducing crashes.
* [A deadlock](https://github.com/m417z/global-inject-demo/commit/608f007e9e0566b340ac2e6684bf2001106cca57) as a result of suspending my own thread (when there's only one, [on Windows versions older than Windows 10](https://blogs.blackberry.com/en/2017/10/windows-10-parallel-loading-breakdown)).

Being injected into all processes, stability is very important, since a bug can affect any process on the system, and a crash can bring any process down (or worse, all of them). Therefore, I've been taking extra care while implementing the DLL, and I try to investigate any problem that's being reported. Recently, I received this report which got me busy for a couple of days...

# The crashing game

A Windhawk user contacted me and reported that one of his games crashed on startup if Windhawk was enabled. I asked for a crash dump, but there was no mention of Windhawk in the stack trace. I wanted to reproduce it locally, but it was a paid game, dozens of gigabytes in size, making it tricky. Luckily, only getting a copy of the exe and several dlls was enough. Without Windhawk, it showed an error message of not being able to find the other files. With Windhawk, the game crashed.

It took me a while to reduce the crash to thread creation - the game crashed with a modified version of Windhawk which just creates a remote thread which does nothing. I went one step further and patched the game's entry point to create a new thread which does nothing, and it also crashed. Why would an empty thread cause a crash?

# Thread initialization and TLS callbacks

When a thread is created with a function such as [`CreateThread`](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread), the caller specifies a pointer to the function to be executed by the thread. But the specified function isn't the first function that the thread runs. Each thread begins its actual execution in the `LdrInitializeThunk` function in `ntdll.dll`. For more details about the `LdrInitializeThunk` function, check out [this blog post](http://www.nynaeve.net/?p=205) by Ken Johnson.

Among other things, the `LdrInitializeThunk` function invokes `LdrpInitializeThread` to perform per-thread initialization, which primarily involves invoking `DllMain` and TLS (Thread Local Storage) callbacks for loaded modules. This reminded me of the stack trace of the crash:

```
...
ntdll!KiUserExceptionDispatch+0x2e
0x56bbba2
...
game!func2
game!func1
ntdll!ImageTlsCallbackCaller+0x1a
ntdll!LdrpCallInitRoutine+0x6b
ntdll!LdrpCallTlsInitializers+0xc5
ntdll!LdrpInitializeThread+0x1f7
ntdll!_LdrpInitialize+0x93
ntdll!LdrpInitializeInternal+0x6b
ntdll!LdrInitializeThunk+0xe
```

The game's TLS callback crashed while trying to call a function via an invalid pointer. It's easy to come up with a simple program which breaks in a similar manner if an extra thread is created. Here's one example:

```cpp
#include <windows.h>
#include <atomic>
#include <iostream>

struct Task {
    int task_id;
    // ...
};

Task g_tasks[] = { {-1}, {101}, {102}, {103} };
std::atomic<int> g_task_index = 0;
thread_local int g_thread_task_id = g_tasks[g_task_index++].task_id;

DWORD WINAPI WorkerThread(LPVOID) {
    std::cout << "Working on task " << g_thread_task_id << "\n";
    return 0;
}

int main() {
    //CreateThread(nullptr, 0, [](LPVOID) -> DWORD { return 0; }, nullptr, 0, nullptr);

    // Run tasks.
    for (int i = 0; i < 3; i++) {
        CreateThread(nullptr, 0, WorkerThread, nullptr, 0, nullptr);
        Sleep(1000);
    }

    std::cout << "Exiting\n";
}
```

The example might be contrived and might violate some rules, but it does demonstrate the extra thread breakage. If you compile and run it, you'll see the following output:

```
Working on task 101
Working on task 102
Working on task 103
Exiting
```

But if you uncomment the commented `CreateThread` call which creates a thread that does nothing, you'll see an output similar to the following:

```
Working on task 102
Working on task 103
Working on task 1079092432
Exiting
```

That's because the extra thread increases the value of `g_task_index`, and the last thread gets a garbage value due to out-of-bounds read.

If you comment back the `CreateThread` call and run Windhawk, you'll see broken output again, which demonstrates how Windhawk can cause a program to stop working correctly. Can Windhawk be fixed to prevent such breakage?

# Looking for a fix

Programs which are incompatible with new threads aren't common, the user can just exclude the game in Windhawk and call it a day. Still, having Windhawk compatible with more programs is a worthy goal, not only because somebody might want to customize such programs, but also because users don't have an indication that Windhawk causes the crash, which might result in wasted time and frustration while trying to understand what's going on.

I looked for a way to avoid calling TLS callbacks to reduce side effects and, in some cases, prevent crashes. I still needed to create a new thread to manage the state of the injected library, so I was studying the thread creation flow and looked for a way to create a new thread while avoiding TLS callbacks.

**P.S.** You could argue that creating a new thread in the target process isn't strictly necessary. I could inject code and run it in the context of an existing thread, e.g. with a [special user-mode APC](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/types-of-apcs). While it might work for a proof of concept, it complicates the implementation, and raises questions such as which thread should be used, how to ensure the target thread doesn't hold an important lock, and what to do if all threads are suspended or in a waiting state.

While looking for information about thread creation, I stumbled upon the [Windows Internals: SkipThreadAttach](https://waleedassar.blogspot.com/2012/12/skipthreadattach.html) blog post by [Walied Assar](https://twitter.com/waleedassar). The blog post discusses a flag that can be passed to the `NtCreateThreadEx` function to create a thread while skipping most of the initialization steps in `LdrpInitializeThread`. Here's the function's partial implementation:

```cpp
VOID NTAPI LdrpInitializeThread(IN PCONTEXT Context)
{
    PTEB Teb = NtCurrentTeb();

    // .NET-specific initialization
    if (UseCOR && (Teb->SameTebFlags & InitialThread))
    {
        // ...
    }

    RtlpInitializeThreadActivationContextStack();

    if ((Teb->SameTebFlags & SkipThreadAttach) && !(Teb->SameTebFlags & RanProcessInit))
    {
        return;
    }

    if (Teb->SameTebFlags & LoaderWorker)
    {
        return;
    }

    // - Allocate TLS
    // - For each module:
    //   - Call TLS
    //   - Call DllMain with DLL_THREAD_ATTACH
    // - For the main module, call TLS
    //
    // ReactOS implementation:
    // https://github.com/reactos/reactos/blob/06b25bc9dd22060aad79f29a97f8d23dce8227e0/dll/ntdll/ldr/ldrinit.c#L527-L632
}
```

As you can see, such thread skips TLS allocation, and triggers no TLS or `DllMain` callbacks. Let's try using it with our test program by replacing the commented `CreateThread` call with a `NtCreateThreadEx` call with the `SkipThreadAttach` flag:

```cpp
    using NtCreateThreadEx_t = NTSTATUS(WINAPI*)(
        _Out_ PHANDLE ThreadHandle,
        _In_ ACCESS_MASK DesiredAccess,
        _In_opt_ LPVOID ObjectAttributes,  // POBJECT_ATTRIBUTES
        _In_ HANDLE ProcessHandle,
        _In_ PVOID StartRoutine,  // PUSER_THREAD_START_ROUTINE
        _In_opt_ PVOID Argument,
        _In_ ULONG CreateFlags,  // THREAD_CREATE_FLAGS_*
        _In_ SIZE_T ZeroBits,
        _In_ SIZE_T StackSize,
        _In_ SIZE_T MaximumStackSize,
        _In_opt_ LPVOID AttributeList  // PPS_ATTRIBUTE_LIST
        );
    NtCreateThreadEx_t pNtCreateThreadEx =
        (NtCreateThreadEx_t)GetProcAddress(GetModuleHandle(L"ntdll.dll"), "NtCreateThreadEx");
    const ULONG SkipThreadAttach = 0x2;
    HANDLE hThread;
    NTSTATUS result = pNtCreateThreadEx(&hThread, THREAD_ALL_ACCESS, nullptr,
        GetCurrentProcess(), (PTHREAD_START_ROUTINE)([](LPVOID) -> DWORD {
            std::cout << "Thread created\n";
            return 0;
        }), nullptr,
        SkipThreadAttach, 0, 0, 0, nullptr);
```

Running the program prints the following output:

```
Thread created
Working on task 101
Working on task 102
Working on task 103
Exiting
```

It works! So, did we find the perfect fix?

# Surviving without TLS

It would have been nice to have a single flag fix everything, but just using `SkipThreadAttach` with Windhawk threads wasn't enough. My goal was to avoid calling TLS callbacks to reduce side effects, but the flag goes one step further and skips TLS initialization altogether. This means that any code running in such a thread must not use TLS, but the injected DLL did. I considered initializing TLS for the DLL manually, but the methods I found were [Windows-version-specific](https://github.com/DarthTon/Blackbone/blob/a672509b5458efeb68f65436259b96fa8cd4dcfc/src/BlackBone/Symbols/PatternLoader.cpp#L104) and [tricky to use](https://github.com/DarthTon/Blackbone/blob/a672509b5458efeb68f65436259b96fa8cd4dcfc/src/BlackBone/ManualMap/Native/NtLoader.cpp#L205). Also (spoiler) TLS usage in the injected DLL was just one problem. What I ended up doing is just getting rid of all TLS usages, replacing them with alternatives:

* `thread_local` variables were replaced with [the FLS API](https://learn.microsoft.com/en-us/windows/win32/api/fibersapi/). I used the [ThreadLocal](https://github.com/wang-bin/ThreadLocal) library that wraps it nicely.
* Static local variables with [thread-safe initialization](https://stackoverflow.com/questions/56458581/why-msvc-thread-safe-initialization-of-static-local-variables-use-tls) were replaced with `std::call_once`, which uses `InitOnceBeginInitialize` and `InitOnceComplete`.

After making sure the injected DLL doesn't use TLS, everything seemed to work. The DLL got injected into all processes successfully, and no process crashed at first. But after a short testing session I saw that Chrome crashed occasionally, Edge crashed even more, and debug symbols failed to load. All these had the same root cause - any code running in the thread must not use TLS, but in these cases, some code did...

# Chrome crash #1 - DLL Load Notification

Chromium [uses DLL load notification](https://github.com/chromium/chromium/blob/e1f324aa681af54101c1f2d173d92adb80e37088/chrome/common/conflicts/module_watcher_win.cc#L184) APIs, `LdrRegisterDllNotification` and `LdrUnregisterDllNotification`, to keep track of loaded modules in the process. The callback is invoked in the context of the thread that is loading or unloading a module, and the callback that Chromium registers makes use of TLS. Do you see the problem here?

Once my thread loads or unloads a module, Chromium's callback is invoked, and the callback crashes while trying to access TLS variables. Preventing the callback from being invoked should prevent the crash, and is unlikely to cause any harm since Chrome has no good reason to be notified about my DLL anyway. It seems to be a part of the third-party module warning/blocking feature, and I'm OK with my DLL not participating in it.

![Chrome's module conflicts page]({{ site.baseurl }}/images/A-guest-in-another-process-a-story-of-a-remote-thread-crash/Chrome_s-module-conflicts-page.png)

I played with the DLL load notification mechanism and found a fairly simple way to [hook and disable callbacks](https://github.com/m417z/LdrDllNotificationHook) in my thread. It indeed seemed to fix this specific crash, but it wasn't the only one.

# Chrome crash #2 - `NtMapViewOfSection` hook

The next crash I saw in Chromium was also during module loading in my thread. To my surprise, Chromium [hooks the `NtMapViewOfSection` function](https://github.com/chromium/chromium/blob/e1f324aa681af54101c1f2d173d92adb80e37088/chrome/chrome_elf/third_party_dlls/hook.cc#L432) to be able to selectively block modules from being loaded. At this point, I started to realize that using a thread without initialized TLS is quite fragile, it's difficult to predict which third party code will be running on it, and any such code may access TLS variables.

# Edge crash - `BaseThreadInitThunk` hook

Just when I was about to give up, I noticed that Edge crashed at a higher rate than Chrome. At first, I thought that it crashed for the same reasons Chrome did, since both browsers are based on Chromium, but a quick check revealed another hook. Turns out that Edge [hooks `BaseThreadInitThunk`](https://twitter.com/m417z/status/1616540533085667350), checks whether the thread entry point is image-backed, and if not, calls a function named `OnThirdPartyThread`. This function, as you might have guessed, uses TLS variables, causing a crash.

![Edge's BaseThreadInitThunk hook]({{ site.baseurl }}/images/A-guest-in-another-process-a-story-of-a-remote-thread-crash/Edge_s-BaseThreadInitThunk-hook.png)

# Debug symbols fail to load

Windhawk has APIs to download and load debug symbols. To implement this functionality, it uses Microsoft's `msdia140.dll` and `symsrv.dll` libraries. These libraries use TLS variables in `DllMain`, but instead of crashing, the module loader swallows the exception and `LoadLibrary` returns an error. Then, when the thread exits, it crashes while trying to invoke a callback that one of the libraries registered (via `FlsAlloc`) but failed to unregister due to the swallowed crash. It took me a while to track it down and reminded me of the [When Even Crashing Doesn't Work](https://randomascii.wordpress.com/2012/07/05/when-even-crashing-doesnt-work/) blog post of Bruce Dawson.

# TLS usage in mods

Mods may also use TLS variables, which will also cause a crash, but unlike Microsoft Visual C++, Clang (which Windhawk uses to compile the mods) doesn't use TLS for thread-safe initialization of static local variables. It still uses TLS for `thread_local` variables, but these are less common, and native TLS usage can be avoided with [the `-femulated-tls` flag](https://clang.llvm.org/docs/UsersManual.html#cmdoption-femulated-tls).

# Conclusions

Injecting code into other processes can be tricky, as any small thing may lead to an unexpected incompatibility. In this case of thread creation, there are only two options - to have TLS callbacks invoked for this thread, or to prevent them from being invoked. Each option may lead to incompatibilities, but after this journey I'm convinced that invoking TLS callbacks is the safer option.

As a result, I decided to make no changes to Windhawk's default behavior at this point, but I added an advanced option to prevent TLS callbacks from being invoked for Windhawk threads. The option can be applied for selected processes by specifying a list of process paths. This way, rare cases like the specific game can at least be made to work with Windhawk via manual tuning.

![Windhawk advanced options process list]({{ site.baseurl }}/images/A-guest-in-another-process-a-story-of-a-remote-thread-crash/Windhawk-advanced-options-process-list.png)
