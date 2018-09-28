---
layout: post
title: Launching Processes on Linux
---

Recently on the [glibc mailing list](https://sourceware.org/ml/libc-alpha/2018-09/msg00088.html) someone asked why **popen** or **system** tend to fail with ```ENOMEM``` on some environments, even when it seems that the system does have enough available memory.

As we discovered in the thread, it is due the fact that glibc **popen** and **system** functions use the tandard UNIX way of creating process by calling **fork** followed by **execve**. The **fork** is usually implemented as a syscall and Linux implements a copy-on-write optimization to speed up execution.  It means that for most cases where the child process just does trivial computation followed by *exec* syscall to spawn a new process, no kernel page allocation copies will be performed.

So does it mean that as long the code does not trigger a memory write in either parent or child (forcing the kernel to actually copy the affected page range) *fork* and *execve* should show good performance? Unfortunately the kernel work required to allocate a new task structure plus the page table replication is not trivial. Profiling with a [simple benchmark](https://github.com/zatrazz/posix_spawn_bench) shows that most of time spent trying to spawn a simple binary (*/bin/true* in this case) is in the kernel space. On a x86_64 machine:


```
$ perf record ./posix_spawn_bench -b fork_exec
[...]
    11.96%  posix_spawn_ben  [kernel.kallsyms]   [k] copy_pte_range
     9.06%  posix_spawn_ben  [kernel.kallsyms]   [k] unmap_page_range
     7.78%  posix_spawn_ben  [kernel.kallsyms]   [k] _vm_normal_page
     3.88%  true             [kernel.kallsyms]   [k] native_flush_tlb_one_user
     3.60%  posix_spawn_ben  [kernel.kallsyms]   [k] release_pages
     2.97%  true             [kernel.kallsyms]   [k] filemap_map_pages
     2.71%  true             ld-2.27.so          [.] do_lookup_x
     2.39%  posix_spawn_ben  [kernel.kallsyms]   [k] free_pages_and_swap_cache
     2.23%  true             ld-2.27.so          [.] strcmp
```

This may vary is bit between architectures, but it still costly. On *powerpc64le* for instance:

```
     8.19%  true             [kernel.kallsyms]  [k] copypage_power7                                  
     5.17%  true             [kernel.kallsyms]  [k] rfi_flush_fallback                               
     5.06%  posix_spawn_ben  [kernel.kallsyms]  [k] get_page_from_freelist                           
     4.69%  posix_spawn_ben  [kernel.kallsyms]  [k] copypage_power7                                  
     3.03%  posix_spawn_ben  [kernel.kallsyms]  [k] rfi_flush_fallback                               
     2.02%  posix_spawn_ben  [kernel.kallsyms]  [k] clear_user_page                                  
     1.57%  posix_spawn_ben  [kernel.kallsyms]  [k] copy_page_range                                  
     1.33%  true             ld-2.23.so         [.] _dl_relocate_object                              
     1.20%  posix_spawn_ben  [kernel.kallsyms]  [k] _raw_spin_lock    
```

And on *aarch64*:

```
    19.82%  posix_spawn_ben  [kernel.kallsyms]  [k] __ll_sc_atomic_add
    16.53%  posix_spawn_ben  [kernel.kallsyms]  [k] __ll_sc_atomic_add_return
    16.34%  posix_spawn_ben  [kernel.kallsyms]  [k] copy_page_range
    16.16%  posix_spawn_ben  [kernel.kallsyms]  [k] unmap_page_range
     3.68%  posix_spawn_ben  [kernel.kallsyms]  [k] __ll_sc_atomic_sub_return
     1.38%  true             [kernel.kallsyms]  [k] strnlen_user
     1.15%  posix_spawn_ben  [kernel.kallsyms]  [k] free_pages_and_swap_cache
     1.12%  posix_spawn_ben  [kernel.kallsyms]  [k] free_pgd_range
     0.96%  posix_spawn_ben  [kernel.kallsyms]  [k] _raw_spin_unlock_irqrestore
```

It gets even worse when the parent or the child does memory write, since it will trigger deep copies in the kernel to actually replicate the pages. 

To try and fix this performance issue, UNIX-like system started to provide the **vfork** syscall. It was standardized by POSIX and it behaves similar to fork with two important differences:

1. The child share all the memory with parent, including all *mmap* segments and the stack (with some exceptions such as memory lock created by mlock or mlockall).

2. Parent execution is halted until the child either call *execve* functions or *_exit*.

Besides these two main differences **vfork** behaves much like **fork**, with child not duplicating the process facilities described in [fork man-page](http://man7.org/linux/man-pages/man2/vfork.2.html). **vfork** indeed requires much less work than fork and has much better scalability (since the work required to duplicate the internal kernel bookkeeping is constant regardless of the process RSS size).

And even with caveats described in the **vfork** man-page, programs do use it due its advantages over *fork*.  Glibc, for instance, implemented the **posix_spawn** with **vfork** until recently to take advantage of *vfork* performance over **fork**.

However since its introduction **vfork** has inherent issues and its subtle semantics make it hard to use as a complete safe replacement of **fork**. Rich Felker has written an extensive post entitled [vfork considered dangerous](https://ewontfix.com/7/) which describes these problems in much detail:

- Signal handlers and vfork: signal handlers are shared between parent and child and the program should take extra care to not modify parent's memory in unwanted or unsafe way.

- Threads and vfork: Linux vfork suspends onl the calling thread, not the whole process. So synchronization may be required to access shared memory. Synchronization itself has its own issues since mostly of POSIX defined interfaces are not asignal-sync-safe.

- setuid and vfork: vfork allows a multithreaded program to have two different processes sharing memory with different privilege levels.

There are additional issues about on how stack is shared between parent and child, where the compiler needs to be aware of vfork specific semantics to avoid spilling data during code generation (GCC has improved its support, but it is not guaranteed that all compilers would have a quality implementation).

To again try and fix these issues and help system without MMU (which fork implementation is either subpar or impossible), POSIX.1-2001 added the [posix_spawn](http://pubs.opengroup.org/onlinepubs/9699919799/functions/posix_spawn.html) interface. It is a much more complex interface than **fork** or **vfork**, however the environment has some freedom regarding its internal implementation and it allows possible future extensions either through new attributes or new file actions.  Glibc initial implementation used the **fork** plus **execve** and later added a **vfork** path (controlled by the GNU extension ```POSIX_SPAWN_USEVFORK```).

However since glibc posix_spawn used either **fork** or **vfork** internally, it suffered from the same issues described (more specifically [BZ#10354](https://sourceware.org/bugzilla/show_bug.cgi?id=10354), [BZ#14750](https://sourceware.org/bugzilla/show_bug.cgi?id=14750), and [BZ#18433](https://sourceware.org/bugzilla/show_bug.cgi?id=18433)). Worse (or better) the **vfork** optimization was only enabled if no flags was used and the old implementation also suffered from unrelated issues as well (such as not reporting child execution errors).

Motivated by the ewontfix post and MUSL implementation (which implements an optimized posix_spawn using clone plus CLONE_VFORK) I decided to improve the glibc one. After some iterations, glibc 2.24 shipped with an improved implementation of posix_spawn for Linux with the following changes:

- It uses the same a similar **vfork** optimization by calling clone with the appropriate flags (CLONE_VM and CLONE_VFORK) and avoids sharing the stack with parent by allocating an explicit stack through mmap.  The only state not explicitly shared between child and parent is the *errno* used on internal syscalls and it is safe since its synchronization is implicit by CLONE_VFORK (since parent's execution is suspended while child executes).

- It ensures that no signal handlers are enabled on child to avoid unwanted or wrong memory accesses from installed signal handlers.  It requires NSIG syscalls to set or reset the handlers, but it is still faster compared to fork operation.

- Thread cancellation is disabled since **posix_spawn** is not defined as a cancellation entrypoint by POSIX.

- Child execution failures are correctly reported to posix_spawn caller. 

The new implementation fixes a lot of issues from previous one and it also shows much better performance and scalability. Using the same benchmark as before, the new **posix_spawn** implementation is much faster than **fork** plus **execve** and even slightly faster than **vfork**:

![](https://raw.githubusercontent.com/zatrazz/zatrazz.github.io/master-jekyll/images/fork_vs_spawn.png)

![](https://raw.githubusercontent.com/zatrazz/zatrazz.github.io/master-jekyll/images/vfork_vs_spawn.png)

The *process RSS size* is measured in MB and *time* is an arithmetic mean of 10000 executions of a simple process (*/usr/bin/true* in this case).  The **posix_spawn** shows similar scalability as **vfork**, it does not suffer from the fork issue when programs with large VMA usage failed with not enough memory (and forced some users to explicitly enable overcommit to fix it) when trying to spawn new processes.

So what is preventing developers from using **posix_spawn** instead of either **fork** and **vfork**? Unfortunately it is because the **posix_spawn** API does not cover all possible scenarios a process might require when spawning new processes.

There is an [interessting discussion about the current limitations of posix_spawn](https://sourceware.org/ml/libc-alpha/2018-09/msg00088.html) and a [proposal for new extensions](http://lists.gnu.org/archive/html/bug-gnulib/2018-09/msg00047.html) (as was done for POSIX_SPAWN_SETSID, for instance, implemented on glibc 2.26). To summarize thekey points:

- **posix_spawn** lacks the ability to change the initial working directory of the new process, as raised in [Austin Group issue #1208](http://austingroupbugs.net/view.php?id=1208).  This is quite useful because the POSIX counterparts chdir and fchdir change the working directory for the whole process, which troublesome for multithreaded applications (an application would need some synchronization to issue these syscalls if the idea is to use them prior **posix_spawn**).

- There is no way to close all inherited file descriptors that are not set to close-on-exec (```O_CLOEXEC```).  Although it can be mitigated by using ```O_CLOEXEC``` as default, there are still occasions where the program cannot guarantee that a file descriptor has the close-on-exec flag, nor can it set the flag atomically in a portable manner. Linux provides the non-portable FIOCLEX through ioctl, but it is not sufficient since it can be changed by another thread prior **posix_spawn** call.

- There is no easy way to clear close-on-exec flag in file actions: once you have a O_CLOEXEC file descriptor, you can not atomically unset it with current posix_spawn file actions. It has been marked as [accepted by Austin Group](http://austingroupbugs.net/view.php?id=411), and it being tracked by [BZ#23640](https://sourceware.org/bugzilla/show_bug.cgi?id=23640) (it will mostly likely to be fixed in the upcoming glibc 2.29).

All these issues prevent some projects from using **posix_spawn** or at least required extra boilerplate code to handle these limitations.  On the mentioned [libc-alpha discussion](https://sourceware.org/ml/libc-alpha/2018-09/msg00088.html), it was hinted that [OpenJDK still uses vfork on linux for JNI](http://hg.openjdk.java.net/jdk/jdk/file/tip/src/java.base/unix/native/libjava/ProcessImpl_md.c) code because of the **fork** memory and performance issues.

So how does one use **posix_spawn** for its over **vfork** where the program still requires to do unsupported file actions?  One option is to narrow down missing APIs and add them on POSIX specification. This is can take a lot of time and may not be available readily on different systems. However this is the path taken in the current glibc implementation, where extensions might render the implementation non-standard and might even conflict with future POSIX specifications, requiring even more complexity to handle compatibility modes.

Another option, which some programs use, is to use a helper program to actually launch the desired binary.  The helper program basically executes some options selected by some arguments (such as move to another folder or close all related files) and uses the rest of the arguments to spawn the desirable program. The drawback is the program will need to deploy and use an external binary, which would require to spawn an extra process adding overhead on process spawning. This adds even more code complexity to handle the extra possible file actions.

A third option is to add another process spawn interface which would work similar to **vfork**, but with the internal guarantees of **posix_spawn** (dissociated stack for child and parent, no shared signal handler, cancellation disabled) and by calling a user-provided callback.  The callback would be responsible to execute any file action or extra computation.  This has the advantage of freeing the process spawn API to define any possible extra operations (like resetting signal handler or opening a file). Extra care would be needed to define which set of functions would be allowed on the callback, since functions which modify shared state may result in undefined behavior (similar to signal handlers). The [thread](https://sourceware.org/ml/libc-alpha/2018-09/msg00165.html) that discusses **system** and **popen** failures also has some insights on this possible API.

# Summary

- Do not use **vfork**: although it provides performance improvements over **fork** it has a lot of subtle caveats.  Also POSIX.1-2008 has replaced it with posix_spawn.

- Use **posix_spawn** instead, unless your usercase requires some unsupported file operations.  Glibc **posix_spawn** is as fast as **vfork** and can internally take care of the subtle issues required to use **clone** with ```CLONE_VFORK``` and ```CLONE_VM```.

- If **posix_spawn** is not sufficient, one option is to use a helper launcher that executes the requires file operation and lunch the final process.

- Both libc implementation and the Austin Group are working towards adding the required extension to **posix_spawn** to cover the expected real world usercases.

# Acknowledgements

- [Siddhesh Poyarekar](https://siddhesh.in/) for help on revising this article.
