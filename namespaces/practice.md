# Namespaces

## PIDs

The `unshare` program (`man 1 unshare`) is a wrapper around the
`unshare` syscall (`man 2 unshare`).

Let's start Python in a new PID namespace by passing (`-p`):

```
unshare -p python3
```

Run this:

```python
import os; os.getpid()
```

It didn't work!  Unsharing a PID namespace only takes affect for children.

Try again, this time asking the `unshare` program to fork (`-f`) a new child before launching Python.

```
sudo unshare -p -f python3
```

The PID should be 1 now.

Run this:

```python
os.system("cat /proc/1/cmdline")
```

Tricky!  This process can see the procfs, but the procfs disagrees about what process ID 1 is.

Run this (in another shell) to see what the top-most PID is of `python3` (besides PID 1):

```
ps aux | grep python3
```

Exit out of the unshare session.

## Mount

Observe that the `cgroup2` FS is mounted:

```
mount | grep cgroup2
```

Start a bash session with a new mount namespace (`-m`):

```
sudo unshare -m bash
```

Run `mount | grep cgroup2` again to verify that it is still unmounted.

Unmount it:

```
umount /sys/fs/cgroup
ls /sys/fs/cgroup # should be empty -- it's just a regular dir now
```

Exit the unshare session and `ls` the directory again.  It's still
there in the primary namespace!  You should couldn't see it in the
namespace that had the unmount.
