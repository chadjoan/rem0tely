## Remotely (client)

```
Usage:
remotely [options] user@host [options] -- <command> [args for command]
remotely feign [options] user@host [options]

This requires the local machine to have an entry similar to this in its
/etc/exports file:

             localhost(insecure,rw,fsid=root,no_root_squash)

The remotely program allows the given `<command>` to be executed using the
CPU resources of another machine, while still (mostly) behaving as if it
were being executed locally.

Below the terms 'local user' and 'remote user' will be used.
The 'local user' is the user executing the whole "remotely" command;
this user is the one that starts (and finishes) the whole process.
The 'remote user' is the 'user' in the above usage description and must be
a valid user on the machine given by 'host'.

A single invocation of remotely should cause this chain of events to occur:
- The local system's NFS service is started, if it wasn't already running.
- A directory of the form /tmp/remotely-YYYY-MM-DDThh:mm:ss-pid is created.
- The root filesystem directory "/" is mounted (with --bind) to that temp dir.
- The exportfs command is run to expose the temp dir over NFS, but only to
    localhost and on a certain (dynamically allocated) port.
- An ssh connection is established using the user@host shown in the usage
    description. This includes a reverse tunnel that exposes the NFS export
    to the host machine.
- The host machine mounts the local machine's NFS export (which is the
    root filesystem) to a temporary directory of the form
    /tmp/remotely-YYYY-MM-DDThh:mm:ss/local_user@local_ip_or_hostname-pid/
    (This path has an extra directory layer with permissions that
    stop other users on the host machine from accessing the mount-point.)
- The remote user chroots into the temporary directory and sets its
    uid and gid to match the local user's uid and gid.
- If a 'command' argument was provided, that command is now executed until
    completion.
- The remote user returns to its original uid and gid, then exits the chroot.
- The (remote) temporary mount-point is unmounted and removed.
- The ssh connection is closed.
- The exportfs command is used to remove the root filesystem share.
- The (local) temporary mount-point is unmounted and removed.
- The local system's NFS service may be stopped, depending on the value of
    the 'nfs-afterwards' argument (if it was provided) and whether or not
    the NFS service was already started before remotely was invoked.


The local user must have a default private key identity (~/.ssh/xxx_id) file
that is authorized to log in as the user on the host machine.  This identity
must also be allowed to open forward ssh tunnels on the host machine, and it
might also be a good idea to set up X11 forwarding if a graphical application
is being ran this way.

The remotely-host program must be in the user's PATH on the host machine.

Security-wise, both the client (local) and host (remote) machines must
be able to trust each other completely before using remotely. The client
machine will be exposing its root filesystem to the host machine, and the
host machine will be allowing the client machine to execute arbitrary
machine code on it.

There may be technicalities that make a program's behavior diverge from what
it would normally do if executed locally. Here are a few examples:
- The host machine has a different kernel configuration and the program
    uses kernel functionality that is either present locally but not
    remotely, or is present remotely but not locally.
- The host machine doesn't have as much memory (RAM) available (unallocated)
    as the local machine.
... others TBD based on observations ...

Options:
     help
     -help
     --help
             Displays this help message.  All other arguments are ignored
             when this is used.

     nfs-afterwards {on,off,same}
     nfs-afterwards={on,off,same}
             This option determines whether the NFS service is left running
             or stopped after remotely finishes executing the remote task.
             The 'same' value will turn the NFS service off only if it
             was already off when remotely began executing.
             The default is nfs-afterwards=same

     no-sudo
             Prevents remotely from invoking itself with the 'sudo' command
             automatically when it runs into permissions problems.

     feign
             Set up everything on the local machine and ssh into the
             host, but do not run a command on the host (the 'command'
             argument can be omitted if this one is passed).  This will
             result in a shell session on the host that can be experimented
             with while the local machine has all NFS mechanisms set up.
             This is useful for debugging and observing remotely's system
             changes during a session.
             If this is used, the 'command' argument will be ignored, as
             well as any of its arguments and options.

     ssh-opts <string>
     ssh-opts=<string>
             Pass the ssh options given in the string to the ssh invocation.
```
