# File Data

## Stat

Used to obtain file metadata. Can be used to get info about a file without opening it

```c
#include <sys/stat.h>

int stat(const char *path, struct stat *sb);
int lstat(const char *path, struct stat *sb);
int fstat(int fd, struct stat *sb);

#include <sys/stat.h>
#include <fcntl.h>

int fstatat(int fd, const char *path, struct stat *sb, int flag);
```

stat(1) displays info about the file pointed to by *file*. By default stat(1) displays values as human friendly: to view the actual values use ```stat -r file```. stat(1) reveals info such as size, block size, number of I-Nodes etc. ls(1) can display most of the information from the struct stat

**Links**: create a link to an existing file or directory (hard or symbolic). Hard links allow multiple filenames to be associated with the same file since a hard link points to the same inode of a given file, while symlinks are special files that refer to other files by name

```sh
# Write hard link
ln [file1] [file2]

# Write soft link
ln -s [file1] [file2]
```

### st_mode Field

- regular: most common, interpretation of data is up to the application
- directory: contains names of other files and pointer to information on those files
- character special: used for certain types of devices, e.g. terminal
- block special: used for disk devices (typically)
- FIFO: used for interprocess communication (pipes)
- socket: used for network communication
- symbolic link: pointers to another file

## UIDs and GIDs

Every process has 6 or more IDs (found in st_mdoe, st_uid, st_gid)

1. real user ID     // who we really are
2. real group ID    // who we really are
3. effective user ID    // used for file access permission checks
4. effective group ID   // used for file access permission checks
5. saved set-user-ID    // saved by exec functions
6. saved set-group-ID   // saved by exec functions

Whenever a file is setuid, set the effective UID to st_uid

Whenever a file is setgid, set the effective GID to st_gid

(st_uid and st_gid always specify the owner and group owner of a file, regardless of whether it is setuid/setgid)

setuid bit is only meaningful if the st_uid field in the struct stat of the executable is different from the effective UID of the user executing the command

**Least Privilege**: a process should only be given the minimum amount of permissions it needs and not more

```c
#include <unistd.h>

/*
 * Sets the real UID, effective UID, and the saved set-UID.
 */
int setuid(uid_t uid);

/*
 * Succeeds only if the arg is the current real UID or the saved set-UID.
 * Only sets effecitve UID.
 */
int seteuid(uid_t);

uid_t getuid(void);
uid_t geteuid(void);
```

After a call to setuid you can no longer regain any previous permissions. This allows temporary elevated
permissions. Useful for giving a program elevated permissions on startup and then revoking them to ensure
security while the program is running. seteuid allows returning to previous permissions

```c
#include <unistd.h>

int access(const char *path, int mode);

int faccess(int fd, const char *path, int mode, int flags);
```

Tests file accessibility on the basis of real uid and gid. This allows setuid/setgid programs to see if the
real user could access the file without it havign to drop permissions to perform the check

Mode can be a bitwise OR of:

- R_OK: test for read permission
- W_OK: test for write permission
- X_OK: test for execute permission
- F_OK: test for existence of file

Each process has an effective UID and a real UID, just like an effective GID and a real GID. If the setuid(setgid) bit is set in the permissions, the effective UID (GID) will become that of the st_uid (st_gid) of the file's struct stat at execution time.

You can switch between them via seteuid(2); setuid(2) sets both irrevocably

The effective UID and group ID are the ones used for file permission checks, but we can check whether the real ID would have permission via the access(2) system call

## st_mode

st_mode encodes the file access permissions. Uses of the permissions are summarized as follows:

- **open**: need execute file on each directory component of the path
- **open with O_WRONLY or O_RDWR**: need read permission
- **open with O_WRONLY OR O_RDWR**: need write permission
- **O_TRUNC**: need write permission
- **create file**: need write and execute permission for the directory
- **delete**: need write and execute on directory; *file does not matter*
- **execute (via exec family)**: need execute permission
  - if a file is written in a compiled language (e.g. Python) then read is required as the interpreter must be used

Which permission set to use is determined in the following order:

1. if effective UID == 0, grant access (irrespective of file permissions)
2. if effective UID == st_uid on file

    2.1 if appropriate user permission bit is set, grant access
    2.2 else, deny access

3. if effective GID == st_gid

    3.1 if appropriate group permission bit is set, grant access
    3.1 else, deny access

4. if appropriate other permission bit is set, grant access, else deny access

root can ignore any permissions

Once UID check is set, the system doesn't care about group permissions. User permissions deny access to any users with effective UID = st_uid

As the order of checks is fixed and important, it is possible to create fine-grained access controls through group membership and carefully set file permissions

## chmod(2) and chown(2)

```c
#include <sys/stat.h>
#include <fcntl.h>

int chmod(const char *path, mode_t mode);   // acts on file and indirect through symlinks
int lchmod(const char *path, mode_t mode);  // acts on symlink itself
int fchmod(int fd, mode_t mode);            // acts on file descriptor
int fchmodat(int fd, const char *path, mode_t mode, int flag);   // handles relative path name
```

mode can be any of the bits from st_mode as well as:

- S_ISUID: set uid
- S_ISGID: set gid
- S_SISVTX: sticky bit (aka "saved text")
- S_IRWXU: user read, write and execute
- S_IRWXG: group read, write and execute
- S_IRWXO: other read, write and execute

Changing permission bits on a file: must be euther euid 0 or euid == st_uid

```c
#incldue <unistd.h>
#include <fcntl.h>

int chown(const char *path, uid_t owner, gid_t group);
int lchown(const char *path, uid_t owner, gid_t group);
int fchown (int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *path, uid_t owner, gid_t group, int flag);
```

Changes st_uid and st_gid for a file. Generally requires euid 0

owner or group can be -1 to indicate that it remains the same

Non-superusers can change the st_gid field if both

- euid == st_uid
- owner == st_uid and group == egid (or one of the supplementary group IDs)

Only root and the owner of a file can change its permissions

Only root may change the owner of a file, but the owner may change the group ownership of a file

## umask(2)

When creating a new file, it will inherit:

- st_uid == effective UID
- st_gid == either:
  - effective GID of the process (most Linux systems)
  - GID of the directory in which it is created (BSD)

```c
#include <sys/stat.h>

mode_t umask(mode_t numask);
```

umask(2) set the file creation mode mask (**default file permission for new files**). Any bits that are on in the file creation mask are turned off in the file's mode. This allows a user to set a default umask. If a program needs to be able to ensure certain permissions on a file, it may need to turn off (or modify) the umask, which affects only the current process
