# Filesystems

## The Unix Filesystem

A hard disk can be divided into smaller **partitions**. On each partition, a **file system** can be created (or not: see swap partition using the raw disk as a place to stash memory when needed). The filesystem organizes **cylinder groups** and provides addressing structures of each in a super block. Each cylnider group contains the actual data blocks (bytes that make up our files) and blocks and inodes set aside for the metadata of files (inode blocks). The file system is replicates the super block in cylinder groups to help prevent corruption. The actual data of files and directories are stored in **inode data blocks** (inode/meta data) and the **file data blocks** (actual file bytes)

Data blocks containing the actual data (file contents) are referenced from the inode

A directory entry is really just a hard link mapping a "filename" to an inode: you can have many such mappings to the same inode

Directories are special "files" containing a list of hard links

- each directory contains at least two entries:
  - ".": this directory
  - "..": the parent directory

**/tmp/** is a seperate file system (memory file system)

### inodes

The inode number in a directory entry must point to an inode on the same file system (no hardlinks across filesystems)

The inode contains most of the information found in the struct stat

Every inode has a link count (st_nlink): shows how many "things" point to this inode. Only if this is 0 (and no process has the open file) are the data blocks marked as available

To move a file within a single filesystem, we can "move" the directory entry (done by creating a new entry and deleting the old one)---> doesn't really move anything just creates a new link (**does not interact with disk blocks**)

- this temporary creates two simultaneous names for the same file before the old directory entry is removed

To move a file across filesystems you must copy the file

**Operations on a directory are independent of the files and their data** due to data storage in file system

## link(2)

```c
#include <fcntl.h>
#include <unistd.h>

int link(const char *path1, const char *path2);
int linkat(int fd1, const char *path1, int fd2, const char *path2, int flags);
```

link(2) creates a hard link to an existing file, incrementing st_nlink in the process

POSIX allows hard links across filesystems but most implementations don't

Only euid 0 can create links to directories (loops in a filesystem are bad)

The link count (st_nlink) keeps track of how many names for a file exist; if this count is 0 and no process has a file handle open for this file, the data blocks may be released

### unlink(2)

```c
#include <fcntl.h>
#include <unistd.h>

int unlink(const char *path);
int unlinkat(int fd, const char *path, int flags);
```

Removes the given directory entry, decrementing st_nlink in the process

If st_nlink == 0, free data blocks associated with file (...unless processes have the file open)

wait-unlink.c shows the available disk space as hardlinks are added and removed

unlink(2) never actually removes files, just deletes links: data blocks are not modified and "deleted" data can oftentimes be refound on the disk if a directory entry is removed. All links must be removed for the filesystem to remove the mapping from inode to data blocks, thereby freeing the datablocks for usage

### rename(2)

```c
#include <stdio.h>
#include <unistd.h>

int rename(const char *from, const char *to);
int renameat(int fromfd, const char *from, int tofd, const char *to, int flags);
```

If from refers to a file:

- if to exists and is not a directory, its removed and from is renamed to
- if to exists and it is a directory, an error results
- must have w+x perms for the directories containing from/to

If from refers to a director:

- if to exists and is an empty directory (contains only . and ..) it is removed and from is renamed to
- if to exists and is a file, an error results
- must have w+x perms for the directories containing from/to
- if from is a prefix of to an error results

Renaming a file on the system filesystem is trivial, but renaming across filesystems and between files and directories requires a little more work

### symlink(2)

```c
#include <stdio.h>
#include <unistd.h>

int symlink(const char *name1, const char *name2);
int symlinkat(const char *name1, int fd, const char *name2);
```

Creates a **symbolic link**: a special file that contains as its data the pathname of another file. Can create symlinks to any type of file (file is a directory, file does not exist, filesystem/device location etc.)

Get information on symlink using the l-family of functions

### readlink(2)

```c
#include <unistd.h>

ssize_t readlink(const char *path, char *buf, size_t bufsiz);
ssize_t readlinkat(int fd, const char *path, char *buf, size_t bufsize);
```

Determines the target of a symbolic link: **buf is not NULL terminated** (must ensure proper handling)

## Directories

```c
#include <sys/stat.h>
#include <fcntl.h>

int mkdir(const char *path, mode_t mode);
int mkdirat(int fd, const char *path, mode_t mode);
```

Creates a new and empty (except for . and .. entries) directory

Access permissions specified by mode and restricted by umask(2) of the calling process

Ownership as previously discussed (st_uid = euid; st_gid = st_gid of directory it is created in, or st_gid = egid)

```c
#include <unistd.h>

int rmdir(const char *path);
```

Removes the given directory if

- the directory is empty (except for . and ..)
- st_nlink is 0 (after this call)
- no other process has the directory open

### Reading Directories

```c
#include <dirent.h>

DIR *opendir(const char *path);
DIR *fdopendir(int fd);

struct dirent *readdir(DIR *dirp);  // Returns: pointer to next entry if OK, NULL on end of directory or error

int closedir(DIR *dirp);
```

**Don't use these as they arent optimal: use fts(3) instead**: opendir(2)/readdir(2) requires read permissions, while opening a file inside a directory requires exec permissions on the directory

The format of directory entries is filesystem and implementation dependent; use readdir(2)/getdents(2)-- see dirent(3)

Type DIR represents a directory stream, an ordered sequence of all directory entries in a particular directory

File descriptor limitations may or may not apply to directory stream: see COMPATABILITY in opendir(2)

***MIDTERM: for directory traversal use fts(3)***

To recursively remove a directory we need both read and exec permissions

### Changing Directories

```c
#include <unistd.h>

char *getcwd(char *buf, size_t size);
```

Get the kernel's idea of our process's current working directory

```c
#include <unistd.h>

char *chir(const char *path);
int fchdir(int fd);
```

The current working directory is set on a per-process basis

Change the process's current working directory

Requires exec permissions on the directory in question

Example: shell that runs command as a fork creates new working directory, runs command, then exits. **Shell built-ins do not do this**

**CD**: must be a shell built-in

## Directory Size

On a traditional Unix File System:

- a directory consists of a number of blocks, often chosen so that they can be written to disk in an atomic operation (i.e. 512 byte blocks)
- the size of a directory is independent of the sizes of the files therein
- the size of the directory is dependent on the length of the filenames it contains
- directories may increase in size, but do not decrease in size

```hexdump -C .```: shows directory file of current directory (should normally use dedicated system calls such as dirent though). Shows bytes on disk that make up the directory:

```sh
00000000 c0 25 00 00 0c 00 04 01  2e 00 00 00 02 00 00 00  |.%..............|
#next line...
```

- c0 25 00 00: four bytes of the inode in HEX
- 0c 00: two bytes of directory entry length (ex: 0c is 12 decimal so the next directory will start at byte 13)
- 04: one bye showing the type of file (4 means directory)
- 01: one byte showing the length of file name (highest value can be FF which is 255)
- 2e 00 00 00: first directory entry (2e is HEX for Ascii's ".")
- 02 00 00 00: second entry (inode #2 --> points to next line in the hexdup)

The last entry spends the entire remainder of the directory (empty space)

Filesystem uses directory entry length to identify when the next directory entry begins. If the file name length of the entry plus padding is shorter than the directory entry than we have space to add a new entry

**If a new file name can fit in this new space...**

- when directory entries are removed the file name is still there. Data is not removed, only the previous directory entry's length is updated to spend the length of the directory we just removed. This is why the directory length did not change: the total sums of lengths did not change

- if a new file is then created, the directory entry who's length was previously expanded is shrunk back down to give space to the new file. The new file's name overwrites saved name of the file previously deleted

**If a longer file name is added than this space available...**

- the former last file of the directory is shrunken and the new longer file takes up the remainder of directory spac
- inodes are reused when new files are created: no guarantee that an inode at one time is the same file as an inode at another

**Really long file names...**

- directory space is extended to the next size increment (etc. 512 bytes -> 1024 bytes)
- if long files are removed and shorter ones added, then they are placed going in the first spot that they can fit

## /etc/passwd

Called a *user database* by POSIX and is found in /etc/passwd, the password file contains the following fields:

|Description|struct passwd member|
|-----------|--------------------|
|username|char \*pw_name|
|hashed password|char *pw_passwd|
|numerical UID|uid_t pw_uid|
|comment|char \*pw_gecos|
|initial working directory|char *pw_dir|
|initial shell|char *pw_shell|

Overview:

- most fields in the password database may be empty
  - empty password field: anybody can log in (bad)
  - empty home directory field: use / instead
  - empty shell field: use /bin/sh instead
- entries may be duplicated:
  - same GID: multiple users in the same primary group (normal)
  - same UID: system applies same permissions for all accounts (rarely used)
  - same username: system will pick one or the other (almost always a mistake)

### Account Weirdnesses

Second account with same ID for root: toor account (Bourne Again super user)

- enables SU login in case shell is broken for default SU root

If a user's initial shell is a command, upon logging in that command is immediately executed and the user logs out

Having two accounts with the same user name is a bad idea (permissions are based on UID and not name)

## getpwuid(2), /etc/group

```c
#include <pwd.h>

// Look up user account info by UID or name
struct passwd *getpwuid(uid_t uid);
struct passwd *getpwnam(const char *name);

// Iterate over password data sequentially
struct passwd *getpwent(void);

void setpwent(void);
void endpwent(void);
```

**/etc/master.passwd**: contains all fields from /etc/passwd plus additional fields (encrypted/hashed password, class, expire etc.)

### /etc/group

Called a group database by POSIX and found in /etc/group, contains the following fields:

|Description|struct passwd member|
|-----------|--------------------|
|group name|char \*gr_name|
|hashed password|char *gr_passwd|
|numerical GID|gid_t gr_gid|
|array of pointers to usernames|char \**gr_mem|

**newgrp(2)**: changes a user to a new primary group and starts a new shell

```c
// Get group information from /etc/group
#include <grp.h>
#include <unistd.h>

struct group *getgrgid(gid_t gid);
struct group *getgrnam(const char *name);

// Iterate over group data sequentially
struct group *getgrent(void);

void setgrent(void);
void endgrent(void);
```

### Other System Databases

|Description|Data File|Structure|
|-----------|--------------------|---------|
|hosts|/etc/hosts|hostent|
|networks|/etc/networks|netent|
|protocols|/etc/protocols|portent|
|services|/etc/services|servent|

### Overview: getpw* and /etc/group

- only euid 0 gets the password hash from /etc/master.passwd
  - use getpwd() to get hash
- groups can have a password: use newgrp(1) to gain access
- iterating over group membership is complex

## atime, mtime, ctime

struct stat contains st_atime, st_mtime, and st_ctime (all time_t type)

- atime: time of last data access
  - updated by any read on any file (triggering expensive and inefficient IO)
  - mount option disables atime updates
    - touch command will still update the ctime (touch is not aware of filesystem mount)
    - mount is known as reltime (relative time): atime is only updated when the mtime or ctime is newer than the atime OR if the atime is older than 24 hours
      - allows filesystem to support atime operations without punishing system through IO
- mtime: time of last data modification (writing)
- ctime: time of last file status change (change in struct stat itself)
  - updated automatically by various syscalls

```c
#include <sys/time.h>

// Requires file ownership and may update non-standard time fields

int utimes(const char *path, const struct timeeval times[2]);
int lutimes(const char *path, const struct timeeval times[2]);
int futimes(int fd, const struct timeval times[2]);
int utimensat(int fd, const char *path, const struct timespec times[2], int flag);
```

If times is NULL, set atime and mtime to current time (write permissions sufficient)
If times is non-NULL, set atime and mtime accordingly (need to be owner)
Either way, ctime is set to current time
Timespec allows for nanosecond precision

## time(3) is an illusion

```c
#include <sys/time.h>
#include <time.h>

time_t time(time_t *tloc);
int clock_gettime(clockid_t clock_id, struct timespec *tp);

// Time formatting
struct tm *gmtime(const time_t *clock);
char *asctime(const struct tm *tm);
```

time(3) returns the value in time of seconds since 00:00:00 January 1 1970 UCT

clock_gettime(3) stores resolution of the clock identified by clock_id into the location specified by res; clock_id CLOCK_REALTIME represents the amount of time (in seconds and nanoseconds) since 00:)) UCT January 1 1970

Convert itme to human readable date:

1. use gmtime(3) to convert time into UTC held in struct tm (described in tm(3))
2. use asctime(3) to return a string in formatted time

### Managing Time that is not the Current Time

```c
#include <time.h>

time_t mktime(struct tm *tm);

char *ctime(const time_t *clock);
```

mktime(3) operates in the reverse direction from gmtime(3)

ctime(3) is like asctime(3) but takes time_t * instead of struct tm *
