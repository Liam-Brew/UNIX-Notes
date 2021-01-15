# Daemons

## Daemon Processes

A daemon process is a process that runs continually in the background and is disconnected from a controlling terminal. They typically offer a specific service and are started at boot/end at shutdown.

Daemon implications:

- do one thing and one thing only
- resource leaks eventually surface
- consider cwd
- limited user-interaction
- use logging for errors

Writing a daemon (daemon(3)):

- clear environment
- fork off parent process
- change file mode mask (umask)
- create a unique session ID (SID)
- change cwd to a safe place
- close or redirect to standard file descriptors
- open any logs for writing

Daemon conventions:

- prevent multiple instances via a lockfile
- allow easy determination of a PID via a pidfile
- include a system initialization script (for /etc/rc.d, systemmd...)
- configuration file convention /etc/name.conf
- re-read configuration file upon SIGHUP
- relay information via event logging, often done using e.g. syslog(3)

```daemon(3)``` is used to convert a program into a daemon
