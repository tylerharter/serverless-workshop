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

## Memory Limit

* start an interactive Python session: `python3`
* get the PID: `import os; os.getpid()`
* add it to a cgroup (reused from before, or create a new one)
* limit memory to 50 MB: `echo 50m > memory.max`
* allocate a 100 MB string in Python: `s = "A" * 100000000`
* should see `Killed`
* run `cat memory.events` -- you should see that the OOM killer had to kill one process (see `oom_kill` line)

## CPU Limit

* start a new Python session
* add it to a cgroup
* launch an infinite loop: `while True: pass`
* in another terminal, observe one core fully used with `htop`
* choose a core you want it to run on, then move it there: `echo CORENUM > cpuset.cpus`
* check htop again to make sure it worked
* limit it to 50% of every 100ms period: `echo "50000 100000" > cpu.max`
* check htop again

## Docker cgroups

Start a Docker container that runs an infinite loop, with 50% of a core:

```
docker run -d --cpu-period=100000 --cpu-quota=50000 python python -c 'while True: pass'
```

You'll see the container ID -- copy it.

`cd` to here, replacing "CONTAINER" with the ID:

```
/sys/fs/cgroup/system.slice/docker-CONTAINER.scope
```

Look at `cpu.max` -- do you see the settings from before?

Run htop to see you're using half a core.

Now modify `cpu.max` to give it a full core, then check again.

## OpenLambda cgroups

Checkout and build OpenLambda: https://github.com/open-lambda/open-lambda

Create a new worker directory:

```
./ol worker init -p cg-test -i ol-min
```

Edit `cg-test/config.json`:
* set `procs` to 25
* in `registry`, replace `cg-test/registry` with `test-registry`

Launch the worker:

```
./ol worker up -d
```

Look at the `fbomb` lambda: `cat test-registry/fbomb/f.py`

Now invoke it a couple times:

```
curl -X POST localhost:5000/run/fbomb -d '{"times": 20}'

curl -X POST localhost:5000/run/fbomb -d '{"times": 30}'
```

Note that you can only create 24 containers given your limit.

Run:

```
cd /sys/fs/cgroup/cg-test-sandboxes/
ls
```

You'll see the cgroups created by OpenLambda ("cg-NUM").
