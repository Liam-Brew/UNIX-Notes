# Logging and Locking

## syslogd(8)

There are three ways to generate log messages:

- via the kernel routine log(9)
- via the userland routine syslog(3)
- via UDP messages to port 514

## syslog(3)

```c
#include <syslog.h>

void openlog(const char *ident, int logopt, int facility);
void syslog(int priority, const char *message, ...);
```

openlog(3) allows us to set specific options when logging:

- prepend ident to each message
- specify logging options (such as LOG_CONS | LOG_PERROR | LOG_PID)
- specify a facility (such as LOG_DAEMON, LOG_MAIL etc.)

syslog(3) writes a message to the system message logger tagged with priority. A priority is a combination of a facility (as above) and a level (such as LOG_DEBUG, LOG_WARNING or LOG_EMERG)

## Non-blocking I/O

Blocking functions review:

- read(2) from files can block (pipes, networks, terminals)
- write(2) to the same sort of files
- open(2) of a device that waits until a condition occurs (for example, a modem)
- pause(3) which purposefully puts a process to sleep until a signal occurs
- certain ioctl(2)s
- certain IPC functions
- file- or record-locking mechanisms

Non-blocking I/O lets us issue an I/O operation and not have it block forever. If the operation cannot be completed, return is made immediately with an error noting that the operating would have blocked (EWOULDBLOCK or EAGAIN)

Ways to specify nonblocking mode:

- pass O_NONBLOCK to open(2): ```open(path, O_RDRW | O_NONBLOCK);```
- set O_NONBLOCK via fcntl(2):

   ```c
   flags = fcntl(fd, F_GETFL, 0);
   fcntl(fd, F_SETFL, flags | O_NONBLOCK);
   ```

## Resource Locking

Ways to lock to ensure only one process has exclusive access to a resource:

- open file using O_CREAT | O_EXCL, then immediately unlink(2) it
- create a "lockfile" - if file exits, somebody else is using the resource
- use of a semaphore

Most locking mechanisms are advisory; they require the cooperation of the processes. Any form of locking carries the risk of a deadlock (code defensively to account for this)

```c
#include <fcntl.h>

int flock(int fd, int operation);

// Returns 0 on success, -1 on error
```

- applies or removes an advisory lock on the file associated with the file descriptor fd
- operation can be LOCK_NB and any one of:
  - LOCK_SH
  - LOCK_EX
  - LOCK_UN
- locks entire file
- see flockfile(3) for locking stdio streams

### Advisory Record Locking

Record locking is done using fcntl(2), using one of F_GETLK, F_SETLK or F_SETLKW and passing a:

```c
struct flock {
    short   l_typle;    // FDRLCK, F_WRLCK or F_UNLCK
    off_t   l_start;    // offset in bytes from l_whence
    short   l_whence;   // SEEK_SET, SEEK_CUR or SEEK_END
    off_t   l_len;      // length, in bytes; 0 means "lock to EOF"
    pid_t   l_pid;      // returned by F_GETLK 
}
```

Lock types are:

- F_RDLCK: non-exclusive (read) lock; fails if write lock exists
- F_WRLCK: exclusive (write) lock; fails if any lock exists
- F_UNLCK: releases our lock on specified range

```c
#include <unistd.h>

int lockf(int fd, int value, off_t size);

// Returns 0 on success, -1 on error
```

*value* can be:

- F_ULOCK: unlock locked sections
- F_LOCK: lock a section for exclusive use
- F_TLOCK: test and lock a section for exclusive use
- F_TEST: test a section for locks by other processes

Locks are:

- not inherited across fork(2)
- inherited across exec(2)
- released upon exec(2) if close-on-exec is set
- released if a process terminates
- released if a filedescriptor is closed (!)

Locks are associated with a **file and process pair**, not a file descriptor

### "Mandatory" Locking

- Not implemented on all UNIX flavors
  ```sh chmod g+s,g-x file```
- Possible to be circumvented using directories
- Basically doesn't work

## Asynchronous and Memory Mapped I/O

### Synchronous, blocking I/O

|Type/Block|Blocking|Non-blocking|
|----------|--------|-------------|
|Synchronous|read(2)/write(2)|read(2)/write(2) under O_NONBLOCK|
|Asynchronous|I/O multiplexing (select(2)/poll(2))|AIO|

### Asynchronous I/O

- semi-async I/O via select(2)/poll(2)
- BSD derived async I/O
  - limited to terminals and networks
  - enabled via open(2)/fcntl(2) (O_ASYNC, F_SETOWN)
  - uses SIGIO and SIGURG

### POSIX AIO

- see aio(7) on NetBSD
- kernel process manages queued I/O requests
- notification of calling process via signal or sigevent callback function
- calling process can still choose to block/wait
- Linux has multiple implementations:
  - glibc aio(7)
  - libaio

### Memory Mapped I/O

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);

// Returns: point to mapped region on success, MAP_FAILED on error
```

- Protection specified for a region:
  - PROT_READ: region can be read
  - PROT_WRITE: region can be written
  - PROT_EXEC: region can be executed
  - PROT_NONE: region cannot be accessed

- flag needs to be one of MAP_SHARED or MAP_PRIVATE, which may be OR'd with other flags (see mmap(2) for details)
