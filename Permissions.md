# Permissions

## Restricting Processes - POSIX. 1e ACLs

### Review

Nature of UNIX being a multitasking multiuser OS implies the need for:

- user privileges
- file permissions
- process ownership
- management of all finite resources

Need to restrict processes and contain applications as resources are limited

Resource limitations:

- use of getrlimit(2)/sysconf(2)
- per-process or per-user limits
- system-wide hard-coded limits
- system tunable configuration options

UNIX Access Semantics based on File Ownership

Filesystem access:

1. if effective-euid == st_uid

    - 1.1 if appropriate user permission bit is set grant access
    - 1.2 else deny access

2. if effective-gid == st_gid

    - 2.1 if appropriate group permission bit is set grant access
    - 2.2 else deny access

3. if appropriate other permission bit is set, grant access, else deny access

Limitations of Unix access semantics:

- a file can only have one group owner
- group membership quickly becomes convoluted
- different (file- and operating-) systems have different limits on the number of groups a user can be a member of
- any modification of group membership requires the sysadmin to make changes (add/remove members, create new groups, ...)

### Access Control Lists

POSIX.1e Access Control Lists (ACLs) perform more fine-grained access control:

- user can specify individuals or groups with different access
- implemented as 'Extended Attributes' in the filesystem and thus require filesystem reports
- ls(1) indicates their presence via a '+' at the end of the permissions string
- requires special tools: getfacl(1), setfacl(2)
- ordering of ACLs is relevant: experiment with different users and groups to verify the impact of the order
- different OS implement POSIX ACLs differently
  - Linux: getfacl(1)/setfacl(1)/acl(5)
  - macOS: chmod(1) to create/manipulate, ls(1) to inspect

## eUIDs, File Flags, Mount Options, securelevels

Examples of ACL usage to control file and directory access:

- necessary to access privileged resources (port binding < 1024, raw sockets for ICMP etc.)
- handling logins (login(1), sshd(8) etc.)
- raising and changing priveleges (su(1), sudo(8) etc.)

### Pitfalls of Changing eUIDs

- setuid programs
  - require careful raising and lowering of privileges only when needed (Least Privilege)
  - rely on correct ownership and permissions (outside control of program)
- su(1)
  - requires password sharing
  - grants all or nothing access
- sudo(8)
  - often misconfigured granting too broad access (ALL:ALL)
  - additional authentication often dropped (NOPASSWD)

### File Flags

```c
#include <sys/stat.h>
#include <unistd.h>

int chflags(const char *path, u_long flags);
int lchflags(const char *path, u_long flags);
int fchflags(int fd, u_long flags);

// Returns 0 on success, -1 on error
```

Your eUID controls access to resources. These can be restricted to access further via file flags:

- UF_APPEND: file may only be appended to (owner or super-user)
- UF_IMMUTABLE: file may not be changed (owner or super-user)
- SF_APPEND: file may only be appended to (super user-only)
- SF_IMMUTABLE: file may not be changed (super user-only)

### securelevels

To prevent eUID 0 from e.g., changing the mount flags, you can employ securelevels:

- superuser can raise the securelevel, only init(8) can lower it
- in other words, lowering requires a reboot
- four securelevels are defined
  - -1 "Permanently insecure mode"
  - 0 "Insecure mode"
  - 1 "Secure mode"
  - 2 "Highly secure mode"
- see secmodel_securelevel(9)

### Summary

- su(1) and sudo(8) can be used to grant others the ability to run commands as another user, but it can be difficult to restrict access
- file flags may restrict certain use; see chflags(1)/chflags(2) on BSD, chattr(1) on Linux
- mount options like noexec, nosuid, rdonly can restrict and protect filesystems per mount point
- to prevent even root from undoing these protections use securelevels

## Restricted Shells, Chroots, and Jails

Another way of restricting what a user can do is to only allow them to execute specific commands, for example via a restricted shell:

- prohibit cd
- prohibit changing e.g., PATH etc.
- prohibit use of commands containing a '/' (i.e., only commands found in the (fixed) path can be executed)
- prohibit redirecting output into files
- beware break-outs via commands that allow invoking other commands

To properly restrict a shell:

- create a new directory, e.g. /usr/local/rbin
- carefully reviewed executables needed, then link them in there
- ensure those commands cannot shell out themselves
- set PATH=/usr/local/rbin
- mark user config files immutable via chflags(1)

### Chroot

Exposes a restricted copy or view of the filesystem to a process via chroot(2)/chroot(8):

- restrict a process' view of the filesystem hierarchy
- restrict commands by only providing needed executables
- must provide full environment, shared libraries, config files, etc.
- combine with null mounts / mount options
- open file descriptors may be brought into the chroot
- processes outside the chroot are visible

### Jail

jail(2) and jail(8):

- enforce a per-jail process view
- prohibit changing sysctls or securelevels
- prohibit mounting and unmounting filesystems
- can be bount to a specific network address
- prohibit modifying the network configuration
- disable raw sockets

Jails effectively implement a process sandbox environment, forming the first OS-level virtualization

## Process Priorities

All processes (including the jail) compete for computer resources

```c
#include <sys/resources.h>

int getpriority(int which, id_t who);
int setpriority(int which, id_t who, int prio);

// Returns 0 on success and -1 on error
```

- default priority is 0
- which is one of PRIO_PROCESSES, PRIO_PGRP, PRIO_USER: who is PID, PGID, or UID
- prio is a value -20 <= prio <= 20
- only the superuser may lower values
- getpriority(2) may return -1; need to inspect errno
- processes can voluntarily self-restrict their resource utilization
  - use nice(1) or renice(8) to adjust the niceness of your process or group (nice is how much a process is willing to share)
  - can't be reverted once set
- priority does not influence CPU placement

## Processor Affinity and CPU Sets

- pinning a process (group) to a CPU can improve performance by e.g., reducing CPU cache misses
- "processor affinity" or "CPU pinning" lets you assign a process to a specific CPU, but other processes may still be placed on that CPU
- processor affinity is inherited by child processes from its parent, but changing a parent's affinity does not affect running children
- CPU sets let you reserve CPUs for specific processes

## Capabilities, Control Groups, Containers

POSIX capabilities for fine-grain control:

- CAP_CHOWN: ability to chown files
- CAP_SETUID: allow setuid
- CAP_LINUX_IMMUTABLE: allow append-only or immutable flags
- CAP_NET_BIND_SERVICE: allow network sockets < 1024
- CAP_NET_ADMIN: allow interface configuration
- CAP_NET_RAW: raw packets
- CAP_SYS_ADMIN: broad system privs

### Linux Namespaces

Partition kernel resources to expose them with granular visibility to processes:

- mnt: mount points
- pid: process ID visibility
- net: virtualized network stack
- ipc: System V IPC visibility
- uts: Unix Time Sharing (different host- and domain names)
- user: user-IDs and privileges
- time: system time
- cgroup: control groups

### Control Groups

Originally termed process containers; allow for:

- resource limiting (e.g., memory limit)
- prioritization (e.g., CPU utilization, disk I/O throughput)
- accounting
- control (e.g., freezing, checkpointing, and restarting)

Provide the following controls:

- cpu: ability to schedule tasks
- cpuset: CPUs and memory nodes
- freezer: activity of control groups
- hugetlb: large page support (HugeTLB) usage
- io: block device I/O
- memory: memory, kernel memory, swap memory
- perf_event: ability to monitor threads
- pids: number of processes
- rdma: remote direct memory access

cgroups are implemented as a virtual file system, often under /sys/fs/cgroup. See cgroups(7) for details

### Containers

Isolated execution environment providing a form of lightweight virtualization:

- use null and union mounts to provide the right environment
- restrict processes in their utilization
- restrict filesystem views
- restrict processes from what they can see
- restrict processes from what they can do
