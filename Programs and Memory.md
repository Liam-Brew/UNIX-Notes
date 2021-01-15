# Programs and Memory

## Memory Layout of a Process

Can print a variable's memory address using the following format:

```c
(void)printf("argc at : 0x%12lX\n", (unsigned long)&argc);
```

Stacks directly increment location based on size, ex:

- first variable is an int (4 bytes) at location 0x7F7FFF8A8254
- next is array of 10 bytes at at location 0x7F7FFF8A8250
- next is another array at 0x7F7FFF8A8240
- ...

## Program Startup

main() is always called at program startup

When one of the exec functions is called, the kernel needs to start the program. A special startup routine called by kernel sets up things for main. argc is the number of command line arguments (including the command itself), argv is an array of pointers to the arguments. It is guaranteed that argv\[argc] == NULL

The entry point of a program is \_start. _start calls main(). The compiler can be told to create a different entry point for a program via the ```-e [method_name]``` option

Startup routine sets up the environment and moves arguments etc. into the right registers for main() to be called

main() returns an int which is passed to exit(3)

Start flowpath: **_start() -> __start() -> exit(main(...))**

## Program Termination

objdump util: disassemble executable file

There are multiple ways for a process to terminate:

- normal termination:
  - implicit return from main
  - explicit return from main
  - calling exit(3)
  - calling \_exit(2) (or _Exit(2))
  - return of last thread from its start routine
  - calling pthread_exit(3) from last thread
- abnormal termination:
  - calling abort(3)
  - termination by a signal
  - response of the last thread to a cancellation request

### exit(3) and _exit(2)

```c
#include <stdlib.h>
#include <unistd.h>

void exit(int status);

void _exit(int status);

int atexit(void(*function)(void)); // Returns 0 on success, -1 on error
```

exit(3) terminates a process. Before termination it performs the following functions in the order listed:

- call the functions registered with the atexit(3) function, in the reverse order of their registration
- flush all open output streams
- close all open streams
- unlink all files created with the tmpfile(3) function

Following this, exit(3) calls \_exit(2). _exit(2) terminates the process immediately (with consequences we will see later)

atexit(3):

- registers a function with a signature of void function(void) to be called at exit
- functions are invoked at exit in reverse order of registration
- same function can be registered more than once
- extremely useful for **cleaning up open files, freeing certain resources** etc.

### Termination Overview

- to implicitly exit(3), return from main. Exit status depends on C standard and last function call
- explicitly exit(3) or _exit(2) at any time
- register exit handlers via atext(3)
- exit without calling exit handlers etc. via _exit(2) or abort(3)

## The Environment

Interact with environment:

```c
#include <stdlib.h>

char *getenv(const char *name);
// Returns value if found, NULL on error

int putenv(char *string);
int setenv(const char *name, const char *value, int overwrite);
int setenv(const char *name);

// Returns 0 on success, -1 on failure
```

### Dynamic Memory Allocation

```c
#include <stdlib.h>

void *malloc(size_t size);
void *calloc(size_t number, size_t size);
void *realloc(void *ptr, size_t size);

// Returns pointer on success, NULL otherwise

void free(void *ptr);
```

malloc(3) allocates size bytes of unitialized memory

calloc(3) allocates number * size bytes of memory, initialized by zero bytes

realloc(3) lets you grow or shring a previously allocated area, posisbly returning a new pointer

See jemalloc(3) for a lot more info

### Environment Recap

Process environment consists of an array of strings as KEY=VAR pairs

Initial environment is set up at the top of the process space and pointed to by the extern char **environ

Third argument char **envp may be passed to main, which also points to that location

Setting variables in the environment via setenv(3)/putenv(3) may lead to shuffling some or all elements of the environ to dynamically allocated memory, but **envp does not get updated**

Build programs defensively as user can set environment variables to nonsense values

## Process Limits and Identifiers

``ulimit(1)```: examine process resource limits (obsolete for library functions)

```c
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *p);
int setrlimit(int resource, const struct rlimit *rlp);

// Returns 0 on success, -1 on error
```

Changing resource limits follows these rules:

- process may change its soft limit to a value less than or equal to its hard limit
- any process can lower its hard limit greater than or equal to its soft limit
- only a superuser can raise hard limits
- changes are per process only (which is why ulimit must be a shell built-in)

### Process Identifiers

```c
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);
```

Process IDs are guaranteed to be unique and identify a particular executing process with a non-negative integer

Certain processes have fixed, special identifiers. They are:

- swapper, sched, idle, or system, process ID 0: responsible for scheduling
- init, process ID 1: bootstraps a UNIX system, owns orphaned processes
- pagedaemon, process ID 2: responsible for virtual memory system (some UNIX systems)

### Overview

Certain aspects of a process's execution are restricted via limits, specified as soft and hard:

- soft limit: can be raised up to a hard limit
- hard limit: only superuser can raise

Resource limits are enforced *per process*

A process further has (at least) a process ID (PID) and a parent process ID (PPID)

## Process Control

```c
#include <unistd.h>

pid_t fork(void);

// Returns: twice(!), 0 to the child, new pid to the parent, -1 on error
```

fork(2) creates a new process. The new process (child) is an exact copy of the calling process (parent) except for the following:

- the child process has a unique process ID
- the child process has a different parent process ID (i.e., the processID of the parent process)
- the child process has its own copy of the parents descriptors
- the child process's resource utilizations are set to 0

Note: no order of execution between child and parent is guaranteed

### exec(3)

The exec() family of functions are used to completely replace a running process with a new process image. They are all front-ends for the execve(2) system call

```c
#include <unistd.h>

int execve(const char *path, char *const argv[], char *const envp[]);

// Does not return, -1 on error
```

exec(3) functions:

- if it has a v in its name: argv's are a vector (const * char argv[])
- if it has an l in its name: argv's are a list (const char *arg0)
- if it has an e in its name: it takes a char * const envp[] array of environment variables
- if it has a p in its name: uses the PATH environment variable to search for the file

Features:

- open file descriptors are inherited, unless the close-on-exec file flag was set
- ignored signals in the calling process are ignored after exec, but caught signals are reset to default
- real UID/GID is inherited: effective UID/GID is inherited unless the executable was setuid/setgid

### wait(3)

wait() suspends execution of the calling process until status information is available for a terminated child process

waitpid() / wait4() allow waiting for a specific process or process group; wait3()/wait4() allow inspection of resoure usage

wait(2) and waitpid(2) macros:

- WIFEXITED(status): true if the child terminated normall; use WEXITSTATUS(status) to get the exit status
- WIFSIGNALLED(status): true if child terminated abnormally (by receiving a signal it didn't catch); use:
  - WTERMSIG(status) to retrieve signal number
  - WCOREDUMP(status) to see if the child left a core image
- WIFSTOPPED(status): true if the child is currently stopped; use WSTOPSIG(status) to determine the signal that caused this

Additionally, wait(2) will block until a child terminates: pass WNOHANG to waitpid(2) / wait(4) to return immediately

Need to wait to prevent zombies (process that has died but continues to be waited for): uses up resources

### Process Control Recap

- all processes not explicitly instantiated by the kernel were created via fork(2)
- fork(2) creates a copy of the current process, including file descriptors and output buffers
- to replace the current process with a new process image, use the exec(3) family of functions
- after creating a new process via fork(2), the parent process can wait(2) for the child process to reap its exit status and resource utilization: failure to do so will create a zombie process until the parent is terminated, at which point init will reap it
