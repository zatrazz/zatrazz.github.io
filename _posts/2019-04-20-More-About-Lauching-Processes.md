---
layout: post
title: More about launching Processes on Linux
---

Recently an interesting paper from Microsoft Research entitled [A fork() in the road](https://www.microsoft.com/en-us/research/publication/a-fork-in-the-road/) advocates that old time [fork](http://man7.org/linux/man-pages/man2/fork.2.html) should be deprecated and replaced by better alternatives.

The following twitter discussion from the Microsoft researcher [post](https://twitter.com/0xabU/status/1115726071364632576) brought some different ideas and strategies used in other Operations Systems, such as Plan9 [rfork](http://man.cat-v.org/9front/2/fork) (which has the option to start a new namespace clean instead of duplicated/shared states as Linux *clone* syscall) or Fuchsia [zx_process_create](https://fuchsia.googlesource.com/fuchsia/+/refs/heads/master/zircon/docs/syscalls/process_create.md) (which creates a blank process which should be started with *zx_process_start*). Also, Uli Drepper and Orran Krieger stirred some [more discussion](https://www.bu.edu/rhcollab/2019/04/11/a-fork-in-the-road/) about the topic.

Both the paper and twitter discussion suggest that a better alternative to spawning a process in Linux is the POSIX.1-2001 **posix_spawn** function (which I [also advocated earlier](https://zatrazz.github.io/Launching-Process)), at least to replace old *fork+exec* usage. The problem, also pointed out by the paper, is **posix_spawn** still lacks some functionalities to fully replace all common uses of *fork+exec*. However, since the **posix_spawn** API also allows extensions it also possible to improve it.

And this exactly what [Glibc](https://www.gnu.org/software/libc/) has done on its latest releases. It has added the following extension to **posix_spawn**:

- **POSIX_SPAWN_SETSID** to create a new session ID for the spawned process ([BZ#21340](https://sourceware.org/bugzilla/show_bug.cgi?id=21340)).

- **posix_spawn_file_actions_addchdir_np** and **posix_spawn_file_actions_addfchdir_np** ([BZ#17405](http://sourceware.org/bugzilla/show_bug.cgi?id=17405)), the *posix_spawn* counterpart of fchdir/chdir that allows to set the new process working directory. 

- Clear close-on-exec for *posix_spawn adddup2* ([BZ#23640](http://sourceware.org/bugzilla/show_bug.cgi?id=23640)) to solve the issue on multi-thread applications which uses close-on-exec as default, and want to hand-chose specifically file descriptor to purposefully inherited into a child process.

## Create a process in a new session

The **POSIX_SPAWN_SETSID** is similar to **setsid** POSIX call, where the newly spawned process is the session leader of this new session, the group leader of the new process group, and has no controlling terminal. It has been recently accepted as a [POSIX enhancement](http://austingroupbugs.net/view.php?id=1044) and as indicated in Austin Group defect tracker, some systems already implement it (such as [QNX](http://www.qnx.com/developers/docs/6.6.0.update/#com.qnx.doc.qnxsdp.nav/topic/bookset.html?topic=%2Fcom.qnx.doc.neutrino.lib_ref%2Ftopic%2Fp%2Fposix_spawn.html) and [Blackberry OS](https://developer.blackberry.com/native/reference/core/com.qnx.doc.neutrino.lib_ref/topic/p/posix_spawnattr_setxflags.html)).

Its usage is quite simple, since is just an **posix_spawn** attribute flag:

```c
posix_spawnattr_t attrp;
posix_spawnattr_init (&attrp);
posix_spawnattr_setflags (&attrp, POSIX_SPAWN_SETSID)
char *const argv[] = { (char *) "pwd", NULL };
/* On  Linux, argv and envp can be specified as NULL,
   however this is nonstandard and nonportable feature.  */
char *const envp[] = { NULL };
pid_t pid;
posix_spawnp (&pid, "pwd", NULL, &attrp, argv, envp);
```

## The new chdir and fchdir file action

This a long standing issue with **posix_spawn** which required either spawning a helper process to change the working directory or implement it on the target program itself. It has been [proposed before](https://sourceware.org/ml/libc-alpha/2010-08/msg00109.html) on *libc-alpha* as RFC, but it did not generate much traction.

It is not a new thing outside glibc also, Solaris 11 [has implemented](https://docs.oracle.com/cd/E86824_01/html/E54766/posix-spawn-file-actions-addchdir-np-3c.html) it also as an extension and Austin Group is moving ahead with the plan to [formalize it in POSIX Issue 8]( http://austingroupbugs.net/view.php?id=1208).

Since it is another file action extension, its usage is quite simple with *Glibc 2.29*:

```c
posix_spawn_file_actions_t fa;
posix_spawn_file_actions_init (&fa);
posix_spawn_file_actions_addchdir_np (&fa, "/tmp");
/* More file action initializations.  */
char *const argv[] = { (char *) "pwd", NULL };
/* On  Linux, argv and envp can be specified as NULL,
   however this is nonstandard and nonportable feature.  */
char *const envp[] = { NULL };
pid_t pid;
posix_spawnp (&pid, "pwd", &fa, NULL, argv, envp);
```

## Clear close-on-exec for posix_spawn adddup2

With POSIX working toward adding [more interfaces to support atomic FD_CLOEXEC](http://austingroupbugs.net/view.php?id=411) **posix_spawn** lacked a proper way to allow a program to clear the flag and to allow the spawned process to inherit the file descriptor.

For instance, if the program intends to use a *pipe* with *O_CLOEXEC* (created by *pipe2* Linux extension or set with *fcntl*) to communicate with the spawned process it can as bellow with *Glibc 2.29*:

```c
pipefds[2];

pipe2 (pipefds, O_CLOEXEC);

posix_spawn_file_actions_t file_actions;
posix_spawn_file_actions_init (&file_actions);
posix_spawn_file_actions_adddup2 (&file_actions, pipefds[1], pipefds[1]);

char pipestr[3 * sizeof (int) + 1];
snprintf (pipestr, sizeof pipestr, "%i", pipefds[1]);

char *const pargv[] = { (char *) prog, (char *) pipestr, NULL };
char *const penvp[] = { NULL };
pid_t pid;
posix_spawn (&pid, prog, &file_actions, NULL, pargv, penvp);
```

On the example above the target program can parse the first input as integer and use it directly as a file descriptor for a subsequent write.

## A real-world user-case: OpenJDK jspawnhelper

The OpenJDK leverages its process creation by using **posix_spawn** on Linux as well.  [Its implementations](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/native/libjava/ProcessImpl_md.c) explains the rationale about using **posix_spawn** instead of *fork+exec*, some Linux libc implementations internals, and some *Glibc* pitfalls in old versions.  However, even for **posix_spawn** its usage is suboptimal: the main process exec a [helper binary](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/native/jspawnhelper/jspawnhelper.c) that does the required pre-exec work and issues the target binary. The helper process itself uses an [internal shared implementation](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/native/libjava/childproc.c) which basically does:

1. Close the parent sides of the pipes.
2. Give the child sides of the pipes the right file descriptor number to match *STDIN_FILENO* and *STDOUT_FILENO*.
3. Close all file descriptors to avoid resource leakage.
4. Change to the new working directory.
5. Finally, spawn the target binary.

All aforementioned steps can be done using Glibc **posix_spawn** extensions, with the exceptions of third step. For first and second step, the [Java_java_lang_ProcessImpl_forkAndExec](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/native/libjava/ProcessImpl_md.c#L579) function create 5 pipes: *in*, *out*, *err*, *childenv*, and *fail* using the POSIX *pipe* function, and a similar result using **posix_spawn** would be:

```c
int in[2], out[2], err[2], childenv[2], fail[2];
pipe (in);
pipe (out);
pipe (err);
pipe (childenv);
pipe (fail);
/* Any other required initialization.  */
posix_spawn_file_actions_t fa;
posix_spawn_file_actions_init (&fa);
posix_spawn_file_actions_addclose (&fa, in[1]);
posix_spawn_file_actions_addclose (&fa, out[0]);
posix_spawn_file_actions_addclose (&fa, err[0]);
posix_spawn_file_actions_addclose (&fa, childenv[0]);
posix_spawn_file_actions_addclose (&fa, childenv[1]);
posix_spawn_file_actions_addclose (&fa, fail[0]);
posix_spawn_file_actions_adddup2 (&fa, in[0], STDIN_FILENO);
posix_spawn_file_actions_adddup2 (&fa, out[1], STDOUT_FILENO);
/* It does not redirect the stream, although adding such option
   would be possible.  */
posix_spawn_file_actions_adddup2 (&fa, err[1], STDERR_FILENO);
posix_spawn_file_actions_addclose (&fa, in[0]);
posix_spawn_file_actions_addclose (&fa, out[1]);
posix_spawn_file_actions_addclose (&fa, err[1]);
```

Note that using Linux **pipe2** with *O_CLOEXEC* simplifies the steps a bit by removing the requirement of closing the dangling pipes end.

The fourth step can be replaced by using the Glibc extension **posix_spawn_file_actions_addchdir_np**. The remaining issue is how to close all file inherited file descriptor to avoid resource leakage. It has been [discussed on Glibc](https://sourceware.org/bugzilla/show_bug.cgi?id=10353) with the suggestion of adding the *posix_spawn_file_actions_addclosefrom_np* file action extension. To summarize the issue, it has pros and cons:

- POSIX direction to avoid resource leakage is to increase *O_CLOEXEC* support and usage by [extending old APIs](http://austingroupbugs.net/view.php?id=411).
- The **closefrom** can be implemented on Glibc, but it would be suboptimal without proper kernel support.  A userland implementation would require to interact over the file descriptor (either by using */proc/self/fds* or by simpler loop), and it is inherent racy on a multithread environment, and incur in performance scalability issues when the file descriptor usage is large.
- The **posix_spawn** extension is possible and might be an option if its usage is widespread enough (which seems the case since both *OpenJDK* and *Python* uses a similar technique).

At least for both OpenJDK and Python, the main issue since they support language extension through plugins (either by *JNI* or Python C modules), and they can not guarantee that all open files are done with *O_CLOCEXE* neither it can set it in a race-free way (since set using *fcntl* in such case is racy in multithread environments).

## Future extensions

As indicated by the paper, **posix_spawn** is still not widely used as a replacement of *fork+exec*. Some users might require additional actions not covered by current standard or libc extensions, such as:

- Process groups (as *setpgrp*).
- List of supplementary group IDs (as *setgroups*).
- Terminal attributes (as termios related functions).
- The root directory (as *chroot*).
- Interval timers (as *setitimer*).
- Resource limits (as *setrlimit*).
- File mode mask (as *umask*).
- Any other state inherited from parent status.

Other specific usages might also prove useful, it will need an analysis case by case whether it justifies a new **posix_spawn** attribute or file action. Other possible directions would be integrated **posix_spawn** as syscall, such as [Solaris 11.4](https://blogs.oracle.com/solaris/posix_spawn-as-an-actual-system-call) and [MacOSX](https://opensource.apple.com/source/xnu/xnu-4903.221.2/bsd/kern/kern_exec.c.auto.html) has done.
