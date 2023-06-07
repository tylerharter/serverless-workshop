# Practice with Control Groups

## Mounting

Make sure there's a cgroup2 mount:

```
mount | grep cgroup2
```

Make another mount:

```
mkdir /same_cg
mount -t cgroup2 none /same_cg
```

Make a first cgroup:

```
cd /same_cg
mkdir top
cd top
ls
```

Note: there's already a bunch of stuff here!  It's a pseudo file
system, and these are not real files (we interface with cgroups by
writing to these).

Let's look at the default mount:

```
cd /sys/fs/cgroup
ls
cd top
```

Note the `top` cgroup is already here.  In cgroups v1, different
mounts could control different processes, but in cgroups v2, every
mount of the pseudo file system is just a view into the same
hierarchy.

## Nested cgroups

Still in `top`, run the following:

```
mkdir cg1
mkdir cg2
cd cg1
ls
```

There's not much here.  Our parent (`top`) hasn't passed control down
to us.  Let's see what we have:

```
cat cgroup.controllers
```

Nothing!  Tell our parent that they should add cpu, memory, and pids to the list of controllers to pass down:

```
echo "+cpu +memory +pids" > ../cgroup.subtree_control
```

Now we have more!  Let's see:

```
ls
cat cgroup.controllers
```

## Join a cgroup

First, start a process, an interactive Python session:

```
python3
```

Get the process ID (PID):

```python
import os; os.getpid()
```

Copy that PID.  **In another terminal...**

```
cd /sys/fs/cgroup/top/cg1/
echo (PASTE PID) > cgroup.procs
```

## Memory Stats

Run `cat memory.current` -- nothing yet!  cgroups aren't great about
memory allocations as things move around.

Allocate about 1 MB in a big string from the interactive Python session:

```python
s = "A" * 1000000
```

Look at `memory.current` again -- should be about 1 MB now.

## Freezer

Let's make the cgroup non-schedulable:

```
echo 1 > cgroup.freeze
```

Try typing something in the session -- it won't work!

You can unfreeze by writing 0 to the file.

## Moving cgroups

As in the last session, freeze it again, and try to type something.
This time, we'll move the process to a different cgroup that's not
frozen.

```
cd ../cg2
echo (PASTE PID) > cgroup.procs
```

The stuff you typed should execute now!

## Deleting cgroups

```
cd ..
rmdir cg1
```

`rmdir` normally only works on empty directories, but it works here
because cgroup2 is a pseudo file system.

What about a cgroup with running processes?

```
rmdir cg2
```

You should see this: `rmdir: failed to remove 'cg2': Device or resource busy`

Kill it, then remove it:

```
echo 1 > cg2/cgroup.kill
rmdir cg2
```

**Important:** the above usually works, but killing a cgroup is
  asyncronous.  When it is slow, the `rmdir` call will fail.  You'll
  need to retry, or `poll` the `cgroup.events` to wait until it is not
  populated.

## Fork Bomb

Pull a test program from OL that attempts to fork a given number of
times (to test that we can limit it with the `pids` cgroup):

```
cd /tmp
wget https://raw.githubusercontent.com/open-lambda/open-lambda/main/test-registry/fbomb/f.py
cat f.py
python3
```

Try forking 100 times:

```python
import f
f.fork_times(100)
```

It should print 100.

Now, in another terminal

```
cd /sys/fs/cgroup/top
mkdir cg1
cd cg1
ps aux | grep python3
```

Add the process to the cgroup and limit to 10 processes (PIDs):

```
echo (PASTE PID) > cgroup.procs
echo 10 > pids.max
```

Run `f.fork_times(100)` in the Python session and note that it only
creates 9 (the limit of 10 includes the already running process).

## Memory: TODO

## CPU: TODO

