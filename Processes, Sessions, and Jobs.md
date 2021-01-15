# Processes, Sessions, and Jobs

## Login Process

init(8): reads /etc/ttys

getty(8): opens terminal, prints "login: ", reads username

login(1):

- getpass(3), hash, compare to getpwnam(3)
- register login in system databases
- read/display various files
- initgroups(3)/setgid(2), initialize environment
- chdir(2) to new home directory
- chown(2) terminal device
- setuid(2) to user's uid, exec(3) shell

Process relations:

> kernel -> init(8)         # explicit creation
> init(8) -> getty(8)       # fork(2) + exec(3)
> getty(8) -> login(1)      # exec(3)
> login(1) -> \$SHELL       # exec(3)
> $SHELL -> ls(1)           # fork(2) + exec(3)

IDs:

> init(8)       # PID1, PPID0, EUID 0
> getty(8)      # PID N, PPID 1, EUID 0
> login(1)      # PID N, PPID 1, EUID 0
>\$SHELL        # PID N, PPID 1, EUID U
> ls(1)         # PID M, PPID N, EUID U

Boot and login process illustrates:

- process creation sequence
- process ownership
- process groups and sessions
- things are generally more complex than we initially think

## Process Groups and Sessions

### Process Groups

```c
#include <unistd.h>

pid_t getpgrp(void);
pid_t getpgid(pid_t pid);

// Returns group-ID, -1 on error (getpgid(2) only)
```

In addition to having a PID, each process also belongs to a process group (a collection of processes associated with the same job/terminal). Each process group has a unique process group ID. Process group IDs (like PIDs) are positive integers and can be stored in a pid_t data type. Each process group can have a process group leader. The leader:
> is identified by its process group ID == PID
> can create a new process group and create processes in the group
A  process can set its (or its children's) process groups using setpgid(2)

### Sessions

```c
#include <unistd.h>

pid_t setsid(void);

// Returns process group-ID if ok, -1 otherwise
```

A session is a collection of one or more process groups. If the calling process is not a group leader, this function creates a new session. Three things happen:

1. the process becomes the session leader in this new session
2. the process becomes the process group leader of a new process group3
3. the process has no controlling terminal

Process Groups:

> ```$ ps -o pid,ppid,pgid,sid,comm | ./cat1 | ./cat2```
> |PID|PPID|PGID|SID|COMMAND|
> |---|----|----|---|-------|
> |265|586|265|265|-sh|
> |296|265|296|265|ps|
> |689|265|296|265|-sh|
> |981|265|296|265|-sh|
> ```$ echo $$```
> 265

### Process Groups and Sessions Overview

- each process belongs to a process group
- a session is a collection of one or more process groups
- process groups are used for distribution of (keyboard generated) signals
- process groups are used to implement job control in a shell (**USEFUL FOR FINAL**):
  - process may have the same process group as the terminal are foreground and may read

## Job Control

- both background and foreground process groups may report a change in status to the login shell
- the foreground process group can perform I/O on the controlling terminal
- the controlling terminal can generate signals via keyboard interrupts to send to the foreground process group
- the background process group may be able to write to the controlling terminal
- the background process group may generate a signal to send to the controlling terminal if it needs to perform I/O
- the shell may move process groups into the foreground or background, suspend or continue them
- we can send any signal to any process via kill(1)

## Signals

- way for a process to be notified of asynchronous events. Some examples:
  - user hits CTRL+C (SIGINT)
  - user hits CTRL+Z (SIGTSTP)
  - background process attempts I/O on the controlling terminal (SIGTTOU / SIGTTIN)
  - a timer you set has gone off (SIGALRM)
  - user disconnected from the system (SIGHUP)
  - user resized the terminal window (SIGWINCH)
  - ...
  - see also signal(2)/signal(3)/signal(7)
- other ways for signals to be generated:
  - terminal generated signals (user presses a key combination which causes the terminal driver to generate a signal)
  - hardware exceptions (divide by 0, invalid memory references, etc.)
  - software conditions (other side of a pipe no longer exists, urgent data has arrived on a network file descriptor, etc.)
  - kill(1) allows a user to send any signal to any process (if the user is the owner or superuser)
  - kill(2), the system call

Once a signal is recieved, we can do several things:

- accept the default. Have the kernel do whatever is defined as the default action for this signal
- ignore it (note: there are some signals which we cannot or should not ignore)
- catch it. Have the kernel call a function which we define whenever the signal occurs
- block it. The delivery of the signal is postponed until it is unblocked

More detailed signal concepts:

- while executing a signal handler, the signal that triggered it is blocked, but other signals may occur
- blocked signals are marked as pending; you can inspect the set of pending signals
- the signal that triggered you entering a signal handler is automatically unblocked upon exit from that signal handler, meaning it will be delivered and you will re-enter the same handler right away
- multiple signals of the same type arriving in sequence while being blocked may be merged
- after fork(2), all signal dispositions and signal masks remain the same in the child as the parent
- the execve(2) system call reinstates the default action for all signals which were caught, but ignored signals continue to be ignored; the signal mask also remains the same

### signal(3)

```c
#include <signal.h>

typedef void (*sig_t)(int);

sig_t signal(int sig, sig_t func);

// Returns previous signal handler if ok, SIG_ERR otherwise
```

func can be:

- SIG_IGN, which requests we ignore the signal signo
- SIG_DFL, which requests that we accept the default action for signal signo
- a pointer to a function to invoke when the signal is received

On some systems/versions, after a signal handler was executed it was reset to SIG_DFL

### sigaction(2)

```c
#include <signal.h>

int sigaction(int sig, const struct sigaction * restrict act, struct sigaction * restrict fact);

// Returns 0 if ok, -1 otherwise
```

signal(3) is nowadays commonly implemented via sigaction(2):

```c
struct sigaction {
  void      (* _sa_handler)(int sig);
  void      (* _sa_sigaction)(int sig, siginfo_t *info, void *ctx);
  sigset_t  sa_mask;
  int       sa_flags;
}
```

### kill(2)

```c
#include <signal.h>

int kill(pid_t pid, int sig);

// Returns 0 on success, -1 otherwise
```

- if pid > 0, then the signal is sent to the process whose PID is pid
- if pid == 0, then the signal is sent to all processes whose process group ID equals the process group ID of the sender
- if pid == -1, then (not POSIX, but BSD and Linux):
  - if euid == 0, the signal is sent to all processes excluding system processes and the process sending the signal
  - else, the signal is sent to all processes with the same uid as the user excluding the process sending the signal
- if pid < -1, then (Linux) the signal is sent to every process in the process group whose ID is -pid
- if sig == 0, no signal is sent, but error checking is performed (i.e., an easy way to test "does pid exist")

### Signals Overview

- signals are notifications of asynchronous events: delivery is similar to a hardware interrupt (current context is saved, a new context is built)
- signals may be allowed to take the default action (SIG_DFL), be ignored (SIG_IGN), caught (signal(3)/sigaction(2)), or blocked (sigprocmask(2))
- arriving signals may be merged, then immediately delivered: different signals may interrupt the current signal handler
- a number of additional options are available: see sigaction(2)

## Reentrant and Interrupted Functions

### Blocking Functions

Some system calls can block for long periods of time (or forever). These include things like:

- read(2) from files that can block (pipes, networks, terminals)
- write(2) to the same sort of files
- open(2) of a device that waits until a condition occurs (for example, a modem)
- pause(3), which purposefully puts a process to sleep until a signal occurs
- certain ioctl(2)s
- certain IPC functions
- file- or record-locking mechanisms

If a signal handler is invoked while a system call or library function is blocked, then:

- the call may be forced to terminate with the error EINTR
- the call may return with a data transfer shorter than requested
- the call is automatically restarted after the signal handler returns

Which of these behaviors occurs depends on the interface and whether or not the signal handler was established using the SA_RESTART flag (see sigaction(2))

Note: much of this is OS specific, and on some platforms only some calls may be restarted, while others always fail when interrupted

### Overview

- only functions that are guaranteed to be async-signal-safe can safely be used in signal handlers
- POSIX guarantees a set of such async-signal-safe functions; different OS (versions) may add others
- even async-signal-safe functions may set errno, so best to save and reset it
- interrupted system calls may fail, return short, or be restarted, again very much subject to variance across OS flavors and versions
