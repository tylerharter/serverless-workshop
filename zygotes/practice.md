# OpenLambda Zygotes

## Using Intrepreter Zygote Only (No Packages)

Bring up a worker and create some lambdas for benchmarking:

```
./ol worker init -i ol-min
./ol worker up -d
./ol bench init
```

Look at the settings related to the import (Zygote) cache:

```
cat default-ol/config.json | grep import
```

You'll see that `import_cache` is "tree", meaning there will be a
single Zygote tree.

The `import_cache_tree` will refer to a file the defines the general
shape of the tree, "default-zygotes-40.json".

Look at this file.  Each dictionary is a node in the tree.  It has a
list of `packages` to be pre-initialized for that node and a list of
children (other nodes).  You can ignore `split_generation` (this is
just commentary about the order in which the node was added to the
tree).

Depending on actual requests, the actual tree of Zygote sandboxes will
be a subset of the tree described in this file at any given time.

The tool for generating the tree is not released to the repo yet
(still experimenting with how to build a good tree).

Run this:

```
./ol bench py1k-par
```

After warming up (sequentially calling each lambda once), it will
invoke lamdas in parallel for 1 minute and report total throughput.

## Other Settings

Try some other experiments: instead of "tree" for `import_cache`, we
can disable it ("") or initialize multile trees ("multitree").  The
problem with "tree" is that if most requests go to the same Zygote in
the tree, it becomes a bottleneck.  "multitree" initializes multiple
independant Zygote trees, and one is randomly selected each time.
More CPUs => more trees.

You can edit config.json, then restart the worker with `./ol worker up -d`.

Alternatively, you can override the config with a command line arg, like this:

```
./ol worker up -d -o features.import_cache=''
```

OR

```
./ol worker up -d -o features.import_cache='multitree'
```

After the new settings, rerun the benchmark.

## Manually Specified Zygote Tree

By default, `import_cache_tree` refers to a JSON file describing the
tree structure.  You can replace `".../default-zygotes-40.json"` with
an inline tree description.

Let's try that now -- start with the simplest possible tree (just a root):

```json
{"packages": [], "children": []}
```

Don't put quotes around it -- that's just for when you have a path.

Run the pandas benchmark (note that "py" is now "pd"):

```
./ol bench pd1k-par
```

It will be very slow because pandas is slow to import and there is not
Zygote for it.  You might want to even kill it before it finished
warmup.

Now paste in this tree:

```json
{"packages": [],
 "children": [
   {"packages": ["numpy"],
    "children": [
      {"packages": ["pandas"],
       "children": []}
      ]
    }
  ]
}
```

Run this:

```
./ol worker down
```

When a worker shuts down, it dumps some stats about Zygote usage.
Open "default-ol/worker.out" and search from the end for "Import Cache
Tree".  There might be one or many matches depending on whether you're
in "tree" or "multitree" mode.

It might look like this:

```
2023/06/10 12:25:23.693572 1    - ROOT
2023/06/10 12:25:23.693576 33     - numpy
2023/06/10 12:25:23.693581 521      - pandas [indirect: numpy]
```

This means the pandas node was used 521 times and its parent (numpy)
was used 33 times.  Every time a Zygote is used, a few hundred KBs get
consumed in the Zygote sandbox (this remains a bit a mystery to be
investigated further).  The `numpy` node is used more than one because
the `pandas` Zygote is occasionally killed (after running out of
memory) and re-created from the `numpy` node.
