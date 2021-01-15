# Sockets

## socketpair(2)

```C
#include <sys/socket.h>

int socketpair(int domain, int type, int protocol, int *sv);

// Returns 0 if ok, -1 on error
```

socketpair(2) creates an unnamed pair of connected sockets in the specified domain, of the specified type, and using the otpinal protocol

Descriptors used in referencing the new sockets are returned in sv\[0] and sv\[1]. The two sockets are indistinguishable

This call is implemented only in the UNIX or "local" domain (PF_LOCAL identifier)

## socket(2) (PF_LOCAL)

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);

// Returns fd if ok, -1 on error
```

socket(2) creates an endpoint for communication and returns a descriptor

Domain specified selects the address or name space of the socket, which selects the protocol family

Type selects the semantics of communication; protocol selects specific rules/formats for this type. In practice selecting the default protocol by specifying 0 is genrally sufficient

Common domains:

|Domain|Description|
|------|-----------|
|PF_LOCAL|local (previously UNIX) domain protocols|
|PF_INET|ARPA Internet protocols|
|PF_INET6|IPv6 protocols|

Common types:

|Type|Description|
|------|-----------|
|SOCK_STREAM|sequenced, reliable, two-way connection based byte streams (TCP): both parties connected at the same time, reliable, delivered receipts|
|SOCK_DGRAM|connectionless, unreliable messages of a fixed (typically small) maximum length (UDP): sent without persistent connection (unreliable, don't know if packet made it)|
|SOCK_RAW|access to internal network protocols and interfaces|

### bind(2)

```c
#include <sys/socket.h>

int bind(int s, const struct sockaddr *name, socklen_t namelen);

// Returns 0 if ok, -1 on error
```

bind(2) assigns a name to an unnamed socket

Binding a name in the UNIX domain creates a socket in the filesystem. This file inherits permissions per the creating process's umask, but that is non-portable

### send(2)

```c
#include <sys/socket.h>

ssize_t send(int s, const void *msg, size_t len, int flags);
ssize_t sendto(int s, const void *msg, size_t len, int flags, const struct sockaddr *to, socklen_t tolen);

ssize_t recv(int s, const void *buf, size_t llen, int flags);
ssize_t recvfrom(int s, void * restrict buf, size_t len, int flags, struct sockaddr *restrict from, socklen_t fromlen);

// Returns 0 if ok, -1 otherwise
```

### Sockets Overview

- create socket using socket(2)
- attach to a socket using bind(2)
- both processes need to agree on the name to use
- these files are only used for rendezvous, not for message delviery
- sockets are represented as file descriptors, so you can use read(2) and write(2)
- dedicated system calls like recv(2) and send(2) etc. offer specific functionality
- after communication, sockets must be removed using unlink(2)

## socket(PF_INET, SOCK_DGRAM, 0) UDP IPv4

Unline UNIX domain names, Internet socket names are not entered into the file system and, therefore, they do not have to be unlinked after the socket has been closed

The local machine address for a socket can be any valid network address of the machine, or it can be the wildcard value INADDR_ANY

Request any ephemeral port by calling bind(2) with a port number of 0

"Well-known" ports (range 1 - 1023) can only bound by euid 0

Determine used port number (or other information) using getsockname(2)

Convert between network byte order and host byte order using htons(3) and ntohs(3) (which may be noops)

UDP is connectionless/unreliable: you can try to send packets without anything listening

tcpdump(8) to observe network traffic

## socket(PF_INET6, SOCK_STREAM, 0) Two-Way Byte Stream IPv6

Conenctions are asymmetrical: one process requests a connection, the other process accepts the request

One socket is created for each accepted request

Mark socket as willing to accept connections using listen(2)

Pending connections are then accept(2)ed

accept(2) will block if now connections are available

Each connection requires a full handshake

## I/O Multiplexing

When handling I/O on multiple file descriptors, we have the following options:

- blocking mode: open one fd, block (possibly forever), then test the next fd
- fork and use one process for each, communicate using signals or other IPC
- non-blocking mode: open one fd, immediately get results, open next fd, immediately get results, sleep for some time
- asynchronous I/O: get notified by the kernel when either fd is ready for I/O

Instead of blocking forever (undesirable), using non-blocking mode (busy-polling is inefficient) or using asynchronous I/O (somewhat limited), we can:

- build a set of file descriptors we're interested in
- call a function that will return if any of the file descriptors are ready for I/O (or a timeout has elapsed)

### select(2)

```c
#include <sys/select.h>

int select(int nfds, fd_set * restrict readfds, fd_set * restrict writefds, fd_set *restrict exceptfds, struct timeval * restrict timeout);

// Returns # of descriptors if ok, -1 on error
```

select(2) tells us both the total count of descriptors that ready as well as which ones are ready:

- nfds: which descriptors we're interested in
- readfds, writefds, exceptfds: what conditions we're interested in
- timeout: how long we want to wait
  - tvptr == NULL: wait forever
  - tvptr->tvsec == tvptr->tvusec == 0: don't wait at all

File descriptor sets are manipulated using the FD * functions/macros

readfds/writefds sets indicate readiness for read/write; exceptfds indicates an exception condition (for example OOB data, certain terminal events)

EOF means ready for read - read(2) will just return 0 (as usual)

pselect(2) provides finer-grained timeout control: allows you to specify a signal mask (original signal mask is restored upon return)

### Multiplexing Overview

- use select(2) to check multiple file descriptors for I/O readiness
- avoid blocking individual file descriptors by handling I/O in a separate process or thread
- alternative or similar interfaces:
  - poll(2)
  - epoll(2) (Linux)
  - kqueue(2) (*BSD)
