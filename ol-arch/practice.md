# OpenLambda Codebase

## Worker Init

* min-image/Dockerfile
* wasm-image/Dockerfile
* src/worker/commands.go: `initCmd`
    * config has one instance per worker process
* src/worker/helper.go: `initOLDir`
    * cleanup previous state
    * Zygotes, config, worker dir, registry
* src/worker/helper.go: `initOLBaseDir`
    * handler (code), host (scratch files, ol.sock), packages (PyPI)
    * resolv.conf, /dev/* files

## Invoke Lambda: Hot Path

Start listening:
* src/worker/server/server.go: `Main`
* src/worker/server/lambdaServer.go: `NewLambdaServer`
    * `LambdaMgr`

Lambda Incoming (req => funcChan => instChan => SOCK => ol.sock => server.py => user code):
* src/worker/server/lambdaServer.go: `RunLambda`
    * `s.lambdaMgr.Get(img).Invoke(w, r)`
* src/worker/lambda/lambdaFunction.go: `Invoke`
    * `f.funcChan <- req`
* src/worker/lambda/lambdaFunction.go: `Task`
    * `req := <-f.funcChan`
    * `f.pullHandlerIfStale()`
    * `f.instChan <- req`
    * policy: look at `inProgressWorkMs` and `desiredInstances`; `f.newInstance()`
* src/worker/lambda/lambdaInstance.go: `Task`
    * `req = <-f.instChan`
    * assume `sb != nil` (hot path), then `sb.Unpause()`
    * `for req != nil` => might serve many requests before pausing again
    * `sb.Client().Do(httpReq)` -- **SEE SANDBOX LAYER**
        * `linst.TrySendError`: will send back over HTTP (first choice), or log it; include details about Sandbox state
    * `default: req = nil` gets us out of the req loop => `sb.Pause()`

SOCK Incoming (other side of `sb.Client().Do(httpReq)`):
* src/worker/sandbox/sockPool.go: `Create`
    * `sockPath := filepath.Join(cSock.scratchDir, "ol.sock")`
    * `cSock.client = &http.Client`
* src/worker/sandbox/sock.go: `Client`
* min-image/runtimes/python/server.py
    * `"/host/ol.sock"`
    * `file_sock = tornado.netutil.bind_unix_socket(file_sock_path)`
* min-image/runtimes/python/server.py: `web_server`
    * `import f` -- user code
    * `app = tornado.wsgi.WSGIContainer(f.app)`
    * `server = tornado.httpserver.HTTPServer(app)`
        * WSGI (Flask, Django, etc): from `f.app`
        * default: `SockFileHandler` wrapper that calls a simple function

## Invoke Lambda: Cold Path

* src/worker/lambda/lambdaFunction.go: `Task`
    * `f.pullHandlerIfStale()` (see **Cold Function Code**)
    * `f.newInstance()` (see **Cold Sandbox**)

### Cold Function Code:

* src/worker/lambda/lambdaFunction.go: `pullHandlerIfStale`
    * `f.lastPull != nil && int64(now.Sub(*f.lastPull)) < cacheNs` only check every few seconds (default 5)
    * `meta.Installs, err = f.lmgr.PackagePuller.InstallRecursive(meta.Installs)` (see **Cold Packages**)
* src/worker/lambda/handlerPuller.go: `Pull`
    * `pullRemoteFile` (`If-Modified-Since`) or `pullLocalFile` (`stat.ModTime()`)

### Cold Packages:

* src/worker/lambda/packages/packagePuller.go: `InstallRecursive`
    * loop until `installs` is empty -- it can shrink (is we install stuff) or grow (as we discover new dependencies)
    * `pp.GetPkg(pkg)`
* src/worker/lambda/packages/packagePuller.go: `GetPkg`
* src/worker/lambda/packages/packagePuller.go: `sandboxInstall`
    * `sb, err := pp.sbPool.Create(nil, true, pp.pipLambda, scratchDir, meta, common.RT_PYTHON)`
        * `pp.pipLambda` is the code from here: `src/worker/embedded/packagePullerInstaller.py`
        * `scratchDir := filepath.Join(common.Conf.Pkgs_dir, p.Name)` -- this is an unusual case: a sub directory in `packages` is exposed as the writeable directory in this container!
    * needs lots of RAM for install: `MemLimitMB: common.Conf.Limits.Installer_mem_mb`
    * `req, err := http.NewRequest("POST", "http://container/run/pip-install", reqBody)`
    * `resp, err := sb.Client().Do(req)`

### Cold Sandbox

* src/worker/lambda/lambdaInstance.go: `Task`
    * `sb, err = f.lmgr.ZygoteProvider.Create` - Zygotes used and successful (see **Cold Zygotes**)
    * `sb, err = f.lmgr.sbPool.Create` - no Zygote path

### Cold Zygotes

* src/worker/lambda/zygote/importCache.go: `Create`
    * `Lookup` -- find node
* src/worker/lambda/zygote/importCache.go: `createChildSandboxFromNode`
    * `cache.getSandboxInNode` - create the Zygote
    * `childSandboxPool.Create` - create leaf sandbox from the Zygote
* src/worker/lambda/zygote/importCache.go: `getSandboxInNode`
    * ideally just `Unpause` (if we already have it)
    * `cache.createSandboxInNode` (need a new one)
* src/worker/lambda/zygote/importCache.go: `createSandboxInNode`
    * might need some packages!  `cache.pkgPuller.InstallRecursive(node.Packages)`
    * `cache.createChildSandboxFromNode(..., PARENT, ...)` (recursive -- need a Zygote to create this Zygote)

## Interfaces

* src/worker/sandbox/cgroups/api.go: `Cgroup` interface
* src/worker/sandbox/api.go: `SandboxPool` and `Sandbox` (full interfaces)
* `SandboxEvent` (function type interface)
* src/worker/sandbox/safeSandbox.go: `safeSandbox` wrapper
    * all (except Client) use locking
    * `dead` is an error.  If any call returns an error, future calls are fine -- they'll just return the error
    * if the wrapped Sandbox is returned, it is also automatically destroyed
    * notifies each `SandboxEventFunc`
* src/worker/sandbox/evictors.go: `SockEvictor`
    * `SOCKEvictor.Event` implements `SandboxEventFunc`
    * sometimes kills Sandboxes to free up memory.  Everything in OL needs to assume any Sandbox can die at any moment
    * the evictor tries to predict paused containers, but we have an async view of the system.  We had to add `DestroyIfPaused` to the `Sandbox` API to avoid closer coordination between the evictor, lambda layer, and Sandbox layer
    * `src/worker/sandbox/memPool.go`
        * memPool enforces admission control
        * Sandbox destruction and pausing increases memory supply
        * Sandbox creation and unpausing decreases memory supply
* src/worker/lambda/zygote/api.go: `ZygoteProvider`
    * `ImportCacheNode`: primary implementation
    * `MultiTree`: collection of `ImportCacheNode`s (useful when too much contention on a single Zygote node)
