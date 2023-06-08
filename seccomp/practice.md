# Practice with Seccomp and Capabilities

## Capabilities

Start an Ubuntu container:

```
docker run -it ubuntu bash
```

Try unmount the cgroup file system:

```
umount /sys/fs/cgroup
```

It should fail.

Start a new container, but add `--cap-add=CAP_SYS_ADMIN` after the `-it` option.

Does it work now?

## Seccomp

Download the default seccomp profile:

```
wget https://raw.githubusercontent.com/moby/moby/master/profiles/seccomp/default.json -O test.json
```

Edit it -- search for the "rmdir" syscall, and delete that line.

Launch with your modified profile:

```
docker run -it --security-opt seccomp=test.json ubuntu bash
```

Try it:

```
mkdir test
rmdir test
```

You should find `rmdir` is no longer permitted.
