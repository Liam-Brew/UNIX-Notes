# IPC

## Interprocess Communication

Message scheduling:

- asynchronous
  - communication takes place without coordination
  - process sends message -> time elapses -> recepient recieves message without sending process coordination
- synchronous
  - sender and recepient do coordinate
  - message is recieved immediately

Direction:

- unidirectional: communication only occurs from one process to another
- bidirectional: communication is back-and-forth between processes

Relation:

- related processes
  - processes share a common ancestor process (shell pipeline)
- unrelated processes
  - signals

Network communication: process on one host talking to another process on another host

Means of two process communication:

- two processes reading from/writing to a file on a shared file system
  - asynchronous
  - bidirectional
  - related processes
  - unrelated processes
  - variable data
- signals
  - asynchronous
  - unidirectional
  - related proceses
  - unrelated processes
- semphamores (locking mechanism): do not use files, instead use kernel communication structures
  - asynchronous
  - unidirectional
  - related processes
  - unrelated processes
  - same host (rely on same resource)
- shared memory
  - asynchronous
  - bidirectional
  - related processes
  - unrelated processes
  - same host
  - variable data
- pipes
  - synchronous
  - unidirectional
  - related processes
  - variable data
- FIFOs (similar to pipes)
  - synchronous
  - unidirectional
  - related processes
  - unrelated processes
  - variable data
- socketpairs
  - synchronous
  - bidirectional
  - related processes
  - variable data
- sockets (basically file descriptors in terms of read/write)
  - synchronous
  - bidirectional
  - related processes
  - unrelated processes
  - network communication
  - variable data

## System V IPC

Three types of async IPC from System V:

- semaphores
- shared memory
- message queues

All three use IPC structures, referred to by an identifier and a key; all three are limited to communication between processes on one and the same host

These structures are not known by name (can't use file descriptors), use:

- msgget(2)
- semop(2)
- shmat(2)
- ipcrm(1)
- ipcs(1)

### Semaphores

Locking mechanism. Used as a counter to provide access to a shared data object for multiple processes. To obtain a shared resource a process does the following:

1. test semaphore that controls the resource
2. if value of semaphore > 0, decrement semaphore and use resource; increment semaphore when done
3. if value == 0 sleep until value > 0

Obtained using semget(2), properties controlled using semctl(2), operations performed using semop(2)

### Shared Memory

Both processes access a shared area of memory (reducing amount of times memory has to be transferred). Fastest form of IPC

Access to shared region of memory often controlled using semaphores so that processes don't overwrite eachother

Commands:

- shmget(2): obtain a shared memory identifier
- shmat(2): attach shared memory segment to a process' address space
- shmdt(2): detach shared memory segment
- shmctl(2): catch-all for shared memory operations

### Message Queues

Linked list of messages stored in kernel space (consumed only in specified order)

- msgget(2): create or open existing queue
- msgsnd(2): add message at end of queue
- msgrcv(2): recieve messages from queue
- msgctl(2): control queue properties

Message itself is contained in user-defined structure (first element must be a long defining message type (see manual)). Example:

```c
struct mymsg {
    long mtype;         /* Message type */
    char mtext[512];    /* Body of message */
}
```

### POSIX Message Queues

mq(3) provides real-time IPC interface similar to System V message queues:

- message queues are identified by a named identifier (no ftok(3) needed)
- message queues may or may not be exposed in the file system (e.g., /dev/mqueue)
- mqsend(3) and mqrecieve(3) allow both blocking and non-blocking calls
- mqsend(3) lets you specify a priority; equal priority messages are queued as a FIFO, but higher priority messages are inserted before those of a lower priority
- mq(3) provides an asyncrhonous notification mechanism: mqnotify(3)

### System V IPC Overview

- async IPC between processes in the same system
- old but not obsolete
- semaphores are useful to guard access to a shared resource
- shared memory allows for fast IPC
- message queues as a service for popular way to implement "pub sub" models:
  - Amazon Simple Queue Service
  - Apache Kafka
  - ...

## Pipes and FIFOs

### pipe(2)

```c
#include <unistd.h>

int pipe(int filedes[2]);

// Returns 0 if OK, -1 on error
```

$ proc1 | proc2

- oldest and most common form of IPC
- usually unidirectional/half-duplex

Use fork(3) to make a copy (child) process of the calling (parent) process

Parent must wait for child before completing

### popen(3)

```c
#include <stdio.h>

FILE *popen(const char *cmd, const char *type);
FILE *popenve(const char *path, char * const argv, char *const envp, const char *type);

// Returns file stream if ok, NULL on error
```

Creates a pipe to execute the command cmd, and returns a stream pointer that can be used to either read/write to that pipe. Specify read/write through either r or w. cmd is passed to /bin/sc -c

popenve(3) combines the exec call with the redirection of the IO stream

### mkfifo(2)

```c
#include <sys/stat.h>
#include <fcntl.h>

int mkfifo(const char *path, mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);

// Returns 0 if ok, -1 on error
```

- "named pipes"
- allows unrelated processes to communicate
- just a type of file - test for using S_ISFIFO(st mode)
- mode same as for open(2)
- use regular I/O operations (open(2), read(2), write(2), unlink(2))

### Pipes and FIFOs Overview

- basis of UNIX strategy on filters and operating on text streams
- pipes require a common ancestor; FIFOs do not
- data written into a pipe is no longer line buffered
- can have multiple readers/writers (PIPE_BUF bytes are guaranteed to not be interleaved)

Behavior after closing one end:

- read(2) from a pipe whose write end has been closed returns 0 after all data has been read
- write(2) to a pipe whose read end has been closed generates SIGPIPE; if caught or ignored, write(2) returns an error and sets errno to EPIPE
