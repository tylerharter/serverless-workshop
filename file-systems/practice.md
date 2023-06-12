# File Systems with Containers

## Setup

Create some files for practice.  We'll be mounting the directories of
`X` and `Y` in different ways over the `test` directory.

Note that `X` and `Y` both have a `b.txt` file (with different
contents), but otherwise these have different files/subdirs.

```
mkdir -p X Y test
mkdir -p X/subdir
echo a > X/a.txt
echo b1 > X/b.txt
echo b2 > Y/b.txt
echo c > Y/c.txt
find
```

## Bind Mounts

### Read Only Bind Mount

Bind the `X` directory to the `test` directory:

```
sudo mount --bind -o ro X test
```

Observe how this shows up in the mount info:

```
mount | grep test
```

Check that they show the same files: `ls test` vs `ls X`.

Observe the read-only nature of the mount:

```
# should fail
echo hi > test/new1.txt

# should work
echo hi > X/new2.txt

# should show new2.txt
ls test
```

### Bind Mounts over Bind Mounts

Lets bind `Y` over the same place where we mounted `X` (the `test` dir).  We won't pass any options this time, and the default is a writeable mount:

```
sudo mount --bind Y ./test
```

Look at `test` and try creating something inside it to make 3 observations:

1. we DO see the contents of `Y` here
2. we DO NOT see the contents of `X` here anymore
3. write can create a new file (e.g., `new3.txt`) inside X now

### Unmounting

We can see both mounts at this location now:

```
mount | grep test
```

Run this, then check the mounts again and `test` dir again:

```
sudo umount test
```

The `test` directory will go back to showing what is in `X` since we
have "uncovered" it by unmounting `Y`.

## Nested Mounts

Run this:

```
sudo mount --bind Y ./test/subdir
```

Observe that we've created a writable mount point on top of a readonly one:

```
# should fail
echo hi > test/new4.txt

# should work
test/subdir/new4.txt
```

## `chroot`

What OS version are you on, perhaps Ubuntu 22.04?

`cat /etc/os-release`

Let's create a directory with an Ubuntu 20.04 deployment, then chroot
to it.

### Step 1: Populate Directory with Ubuntu 20.04 Files

See notes here: https://iximiuz.com/en/posts/docker-image-to-filesystem/

Pull image using Docker:

```
sudo docker pull ubuntu:20.04
```

Create a container from it:

```
sudo docker create ubuntu:20.04
```

COPY the ID.

Paste and run:

```
sudo docker export PASTE_HERE > root.tar
```

Let's extract that to a directory:

```
mkdir -p root
tar -xf root.tar -C root
```

Look inside the `root` directory.  It's the stuff would normally see
in your FS if you were running Ubuntu 20.04.

### Step 2: Make that directory your root

Run this:

```
sudo chroot ./root
```

Check how paths refer to files with different contents now:

```
`cat /etc/os-release`
```

Some things will work, but there will also be a bunch missing because
usually there are other mounts on top of a root FS.  Some examples:

* `ls /tmp` - not a tmpfs
* `ls /dev` - no devices appear here
* `ls /proc` - procfs not mounted

## Union File Systems

Docker uses union file systems.  Create a container:

```
sudo docker run -d ubuntu sleep infinity
```

Run `mount | grep overlay2`, and you'll see the union mount used for
the container.

The lower layers are colon (":") separated -- you can `ls` these to see what is in each.

Let's try making our own overlay mount with the `X` and `Y` directories from earlier.

We'll let X and Y be readonly.  We'll make three new dirs:
* Z: for new files (or edited from lower layers)
* scratch: overlay FS needs this for bookkeeping
* merged: where we'll have a combined view of X, Y, and Z

```
mkdir -p Z scratch merged
```

Let's mount it (if there are errors, run `sudo dmesg` for more info):

```
sudo mount -t overlay -o lowerdir=X:Y,upperdir=Z,workdir=scratch none merged
ls merged
```

Observe that you can a.txt, b.txt, and c.txt (neither X nor Y has all three, so this is a merged view).

What is in `merged/b.txt`?  X is first in the colon separated list, so
its version wins (`b1`).

Unmount (`sudo umount merged`) and remount with `Y` first, then cat
again (should be `b2` now).

Let's modify `a.txt` by appending:

```
echo AAAA >> merged/a.txt
```

Look at each of the following:

```
cat merged/a.txt
cat X/a.txt
ls Z
cat Z/a.txt
```

We can see the file was copied from X to Z before being modified.

## Mixing and Matching Packages

Let's make 3 Python virtual environments:

```
python3 -m venv ./p1
python3 -m venv ./p2
python3 -m venv ./p3
```

Activate the first one, then install matplotlib:

* `source p1/bin/activate`
* `pip3 install matplotlib`
* `deactivate`

Now do similar steps to install pandas in p2.

Let's efficiently get matplotlib AND pandas in p3.

* `rm -r p3/lib/python3.10/site-packages/*`
* `sudo mount -t overlay -o lowerdir=p1/lib/python3.10/site-packages:p2/lib/python3.10/site-packages,upperdir=Z,workdir=scratch none p3/lib/python3.10/site-packages`
* `source p3/bin/activate`
* `python3`
* `import matplotlib, pandas`

Both work, yay!
