# File I/O

## File Descriptors

A file descriptor (or file handle) is a small, non-negative integer which identifies a file to the kernel. This abstracts the file and lets the system call not have to worry about if its an actual file, pipe, or a socket (for example). A file descriptor can be operated on, passed into functions, passed to children, and used to work with APIs for files instead of networks. stdin, stdout and stderr are 0, 1, and 2 respectively. This is bad to use though, instead uses STDIN_FILENO, STDOUT_FILENO, and STDERR_FILENO

UNIX systems need to be careful of using too much resources on a single process. Therefore, a process is limited in the amount of files it can interact with. To see this, view openmax.c in Week 02. This program determines if there is a difference between what OPEN_MAX is defined as and what getconf OPEN_MAX is by checking the file openings manually

 The ```#ifdef OPEN_MAX```, for example, includes code based on the condition of if OPEN_MAX is defined or not

Because sysconf modifies errno, it is important to explicitly set errno to 0. Otherwise it is possible that a function call prior to calling sysconf sets errno to 0, and we wouldn't be able to tell if the value was not found

getrlimit gets the maximum system resource consumption, which in this case is the maximum number of open files

openFiles is a function that first attempts to identify which file descriptors are currently opened. This iterates over the full set of files and tests each one. The parameter num is the maximum number of files that can be open. File descriptors are tested to see if they are open by using the fstat function. Next, files are opened in a loop which only exits if the limit is encountered, which is indicated by setting errno to EMFILE

Different operating systems may used different fixed constants for resource maximums

### ulimit

**Hard Limit**: a limit that cannot be increased by a non-root user
**Soft Limit**: a limit that a non-root user can increase up to the hard limit. If a non-root user changes a limit such as nproc using ulimit, this will only effect that process and any child processes that process creates after changing the limit

***Question***: From what I can tell, if you set 'ulimit -n unlimited' as root, that means that the users can use the entre limit resource limit allowable for that implementation. This seems like it should be disabled by default and only given to necessary individuals such as developers

### Lessons

Answer to "how many file descriptors can a process have open?": it depends!

- you can't always rely on values being defined for you
- the defined value may not actually apply to your process
- constants required by the standard may, while present, not actually be useful
- use sysconf(3)/getrlimit(2) for runtime values, but keep in mind that the result may change from one implementation to the next
- get in the habit of writing code to verify/check your understanding
- testing across Unix versions can help illustrate the difference

_POSIX_OPEN_MAX is the POSIX standard for the minimum number of files that a process can have open at any time. It basically guarantees on any POSIX platform that any process can have at least up to_POSIX_OPEN_MAX files opened, although this is typically more based on the specific implementation

## open(2) and close(2)

### open(2)

Almost all File I/O can be performed using these five functions:

- open(2)
- close(2)
- read(2)
- write(2)
- lseek(2)

These are mostly system calls wrapped with useful library functions

Files can be created using the following creat command:

```C
#include <fcntl.h>

int creat(const char *path, mode_t mode);
```

creat(2) returns a file handle in write-only mode. This was mad obsolete by open(2). creat() is the same as ```open(path, O_CREAT | O_TRUNC | O_WRONLY, mode)```. open is **atomic**, meaning that all operations complete without the possibility of another process changing state during the operation

```C
#include <fcntl.h>

int open(const char *pathname, int oflag, ... /* mode_t mode */);
```

open() does not create a file descriptor as it returns a file descriptor

the oflag must be one (and only one) of:

- O_RDONLY: open for reading only
- O_WRONLY: open for writing only
- O_RDWR: open for reading and writing

and may be OR'd with any of these:

- O_APPEND: append on each write
- O_CREAT: create the file if it doesn't exist; requires mode argument
- O_EXCL: error if 0_CREAT and file already exists (atomic)
- O_TRUNC: truncate size to 0
- O_NONBLOCK: do not block an open or for data to become available
- O_SYNC: wait for physical I/O to complete

open(2) may fail for the following reasons:

- EEXIST: O_CREAT | O_EXCL was specified, but the file exists
- EMFILE: process has already reached the max number of open file descriptors
- ENOENT: file does not exist
- EPERM: lack of permissions

**Therefore, you must always check the return value of an open call before moving on.** Example of good code:

```C
if (fd = open(path, O_RDWR) < 0){
    /* error */
}

/* do stuff with fd */
```

Can also utilize openat(2) to handle relative pathnames from different working directories in an atomic fashion. Here, pathname is determined relative to the directory associated with the file descriptor fd instead of the current working directory

```C
#include <fcntl.h>

int openat(int drfd, const char *pathname, int oflag, ... /* mode_t mode */);
```

### close(2)

```C
#include <unistd.h>

int close(int fd);
```

Closing a file descriptor releases any record locks on that file. File descriptors not explicitly closed are closed by the kernel when the process terminates. **To avoid leaking file descripotrs, always close(2) them within the same scope**. It is best to do this immediately after you write the open call so you don't forget to close it. You may want to cast close() to void at times: ```(void)close(fd)```. This shows that you are not blatantly ignoring the return value, but since close() doesn't fail on many occassions it is ok to cast it to void

openex.c illustrates opening and closing files with various options such as truncating

## read(2), write(2), lseek(2)

### read(2)

```C
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t nbytes);
```

read() begins reading at the current offset and increments the offset by the number of bytes actually read. It is the code's author to ensure that buf is big enough to handle nbytes (otherwise a segfault happens). There can be several cases where read returns fewer than the number of bytes requested. For example:

- EOF reached before requested number of bytes have been read
- reading from a network, buffering can cause delays in the arrival of data
- record-oriented devices (magtape) may return data one record at a time
- interruption by a signal

### write(2)

```C
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t nbytes);
```

write() attempts to write the number of bytes but may not succeed. Like read(), this doesn't necessarily indicate an error. For regular files, write begins at the current offset (unless O_APPEND has been specified, in which case the offset is first set to the end of the file). After the write, the offset is adjusted by the number of bytes actually written. Writing to a file changes its contents. Writing at an offset will overrite any data that happens to be there, and will extend the file if we are writing at EOF

### lseek(2)

```C
#include <sys/types.h>
#include <fcntl.h>

off_t lseek(int fd, off_t offset, int whence);
```

Offset is similar to a cassette tape: read/write at a location (play/record) or move offset (fast forward/rewind)

The value of whence determines how offset is used:

- SEEK_SET bytes from the beginning of the file
- SEEK_CUR bytes from the current file position
- SEEK_END bytes from the end of the file

Can also do "weird" things using lseek(2):

- seek to a negative offset (rewinding)
- seek 0 bytes from the current position: ex: useful for seeking beginning or end (SEEK_SET with 0)
- seek past the end of the file
  - holes can be in files: large portions of a file that contains null characters and is not stored in any data block (created through direct access to further memory space). When lseek is used to go beyond the end of a file, the space between is filled with null bytes

lseek(2) does not incure I/O on the disk and is therefore constant time

### I/O Efficiency

The smaller a buffer size is, the more iterations are required, and the more calls to read/write are performed. The larger the buffer size the more efficient the program is

Can't keep gaining efficiency by continuing to increase buffer size beyond a certain point due to the file system. The file system has a fixed block size in which it reads data from the disk, and no matter how large the buffer is the filesystem cannot read more efficiently than that block size is

Way to determine optimal I/O size (block system size) of a given file: run stat on a file, etc. ```stat -f "%k" tmp/file1```

## File Sharing

As UNIX is a multi-user/multi-tasking system, it is useful if more than one process can act on a single file simultaneously. In order to understand how this is accomplished, we need to examine some kernel data structures which relate to files

- each process table entry has a table of file descriptors, which contain:
  - file descriptor flags (e.g. FD_CLOEXEC, see fcntl(2))
  - a pointer to a file table entry
- the kernel maintains a file table; each entry contains:
  - file status flags (O_APPEND, O_SYNC, O_RDONLY, etc.)
  - current offset
  - pointer to a vnode table entry
- a vnode structure contains
  - vnode information
  - inode information (such as current file size)

After each write(2) completes, the offset in the file table entry is incremented. If the current file offset is larger than the current file size, we change the current file size in i-node table entry

If the file was opened with O_APPEND, set the corresponding flag in file status flags in file table. For each write, the current file offset is first set to the current file size from the i-node entry

lseek(2) merely adjusts the current file offset in file table entry. To seek to the end of a file, just copy the current file size into current file offset. To seek to the beginning of the file, simply set the offset to 0. lseek operates entirely on the meta information and does not need to go to the disk

If multiple processes works on the same thing, ordering can be disturbed and this will cause errors. Processes can interrupt eachother, and changes in the offset between processes can result in corrupted data when it comes to writing and appending. O_APPEND solves this problem for writing to the end, but for writing atomically anywhere in the file use the following:

```C
#include <unistd.h>

ssize_t pread(int fd, void *buf, size_t nbytes, off_t offset);

ssize_t pwrite(int fd, const void *buf, size_t nbytes, off_t offset);
```

In both of these functions the total offset is not changed

### Output Redirection of Multiple Streams

Both stdout and stderr are connected to the terminal. To redirect error messages to /dev/null use the

```sh
ls -l file /nowhere 2>/dev/null
```

wherein 2 is the file descriptor of stderr. To redirect the output, use

```sh
ls -l file /nowhere >file
```

(no file descriptor number is required). For redirecting stdout, notice how when examining file it is listed as 0 bytes, but upon using ```ls -l``` on the file it is non-zero. This is because the redirect opens the file with O_TRUNC, before the ls command is executed. This truncates the file, and when ls looks at it is 0 bytes in file

In redirecting stderr to the same file as stdout using

```sh
ls -l file /nowhere >file 2>file
```

stderr is written first but then stdout overwrites it and

```sh
-rw-r--r--  1 lbrew  users  0 Sep 13 14:37 file
```

is returned. Alternatively, using

```sh
ls -l file /nowhere /does-not-exist >file 2>file
```

with the additional error, a longer error message is returned:

```sh
-rw-r--r--  1 lbrew  users  0 Sep 13 14:34 file
apue$ ls -l file /nowhere /does-not-exist >file 2>file
```

This confirms that both redirections of stdout and stderr open the file in O_TRUNC mode, meaning the offset is 0. Text is overwritten as a result of the two process. O_APPEND (indicated by the redirect >>) **can not** solve this as the order in which processes execute can change. Generally, stderr messages are unbuffered and stdout are buffered

To ensure stderr and stdout go to the same place, don't have both go to a file. Have stderr go to a file and have stderr go to wherever stdout goes. This is through the following:

```sh
ls -l file /nowhere /does-not-exist >file 2>&1 | nl
```

and yields the following output:

```sh
ls: /does-not-exist: No such file or directory
ls: /nowhere: No such file or directory
-rw-r--r--  1 lbrew  users  0 Sep 13 14:50 file
```

The most common use case is to have the shell write both stdout and stderr into a pipe. To open whatever stdout is, use dup:

```C
#include <unistd.h>

int dup(int fd);

int dup2(int fd, int fd2);
```

dup duplicates an existing file descriptor and returns a second file descriptor pointing to the same file table entry as the first one. Since both file pointers point to the same table entry, they share attributes such as file status flags

dup2 allows for redirection of an existing file descriptor. The existing file descriptor is closed and will instead point to the original file table entry. redir.c demonstrates dup2's redirection

### File Descriptor Control

```C
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* int arg */);
```

fcntl(2) is a "catch-all" function that has many uses. They primarily relate to changing properties of an already open file. Some are:

- F_DUPFD: duplicate file descriptors
- F_GETFD: get file descriptor flags
- F_SETFD: set file descriptor flags
- F_GETFL: get file status flags
- F_SETFL: set file status flags

**Synchronous Output**: waits for I/O to be flushed to disk after each call
**Asynchronous Output**: allows OS to handle this more efficiently and cache I/O a bit

To view the differences in times, examine sync-cat.c. Generate a 10MB file through the following:

```sh
dd if=/dev/zero of=file bs=$((1024*1024)) count=10
du -h file
```

Compare times by running ```time ./a.out <file >out``` twice, changing the flags variable to the desired mode for each run. Synchronous mode guarantees that I/O is flushed right away but will suffer a performance penalty when compared to asynchronous operation. The modes can be toggled using fcntl() using the appropriate file flags

```C
#include <unistd.h>     /* System V */
#include <sys/ioctl.h>  /* BSD and Linux */

int ioctl(int fd, int request, ...);
```

Another catch-all function, this is designed to handle device specifics that can't be speecified via any of the previous function calls. Examples include terminal I/O, magtape access, socket I/O, etc.

File descriptors can be represented as files. This allows us to write a program that processes both files as well as standard input. Example of the order of operations:

```sh
apue$ echo one > first
apue$ echo three > third
apue$ echo two | cat first /dev/stdin third
one
two
three
```

```/dev/stdin``` can also be represented as ```/dev/fd/0```
