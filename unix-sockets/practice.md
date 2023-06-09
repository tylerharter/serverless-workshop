# UNIX Domain Sockets

## Root-Owned File

Run:

```
sudo touch out.txt
ls -la
```

Note that it is owned by the root user, and only the owning user can write to it.

## Server

Create a server.py:

```python
import os, socket
s = socket.socket(socket.AF_UNIX)
s.bind("ol.sock")
s.listen()

while True:
    conn, addr = s.accept()
    msg = socket.recv_fds(conn, 1024, 1024)
    print(msg)
    with os.fdopen(msg[1][0], "a") as f:
        f.write("got msg " + str(msg[0], "utf-8") + "\n")
```

Run it as a regular user (NOT root).

This server receives file descriptors (FDs) over a UNIX domain socket and writes strings to those FDs.

If you restart it, you'll need to manually `ol.sock`.

## Client

Create a client.py:

```python
import socket, sys
conn = socket.socket(socket.AF_UNIX)
conn.connect("ol.sock")
f = open("out.txt", "a")
print("sending fd", f.fileno())
text = " ".join(sys.argv[1:]) if len(sys.argv) > 1 else "hi"
msg = socket.send_fds(conn, [bytes(text, "utf-8")], [f.fileno()])
```

Run this as root:

```
sudo python3 client.py hello world
```

This opens `out.txt`, which is owned by the root user.  That's OK, because we're using sudo.

Then it sends the FD for `out.txt` to the server.  The server could
never directly open this file for writing since it is not priviledged,
but the server CAN write to the FD we send from the privileged client.

Look at `out.txt` and confirm the server append the message.
