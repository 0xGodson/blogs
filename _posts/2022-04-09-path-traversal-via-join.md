---
layout: post
title: Exploting Path Traversal in Python Application via os.path.join
subtitle: Path Traversal | LFI
cover-img: /assets/img/wsc.jpg
thumbnail-img: /assets/img/wsc.jpg
share-img: /assets/img/wsc.jpg
tags: [Web]
---

# NullCon CTF 2022 - Sourcer


>Chal URL: http://52.59.124.14:10000/

# Website :

<img src="https://i.imgur.com/9Kj8i21.png">

`Listing files in /usr/src/app/files/:`

<br>

## /app.py will give the Source Code of this Challenge.

>## /flag.txt:

<img src="https://i.imgur.com/Ad3OCNW.png">

`Flag is not here, it is in ../`

We Need to Perform a Path Traversal to Get One Directory Back to Read the Flag.

# Source Code :

```python

#!/usr/bin/python
import http.server
import threading
import socketserver
import re
import os
from io import StringIO

BASEPATH="/usr/src/app"

class FileServerHandler(http.server.SimpleHTTPRequestHandler):
    server_version = "Fil3serv3r"

    def do_GET(self):
        self.send_response(-1337)
        self.send_header('Content-Length', -1337)

        s = StringIO()

        path = self.path.lstrip("/")
        counter = 0
        while ".." in path:
            path = path.replace("../", "")
            counter += 1
            if counter > 10:
                s.write(f"No")
                self.end_headers()
                self.wfile.write(s.getvalue().encode())
                return

        fpath = os.path.join(BASEPATH, "files", path)

        s.write(f"Welcome to @gehaxelt's file server.\n\n")
        if len(fpath) <= len(BASEPATH):
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Hm, this path is not within {BASEPATH}")
        elif os.path.exists(fpath) and os.path.isfile(fpath):
            self.send_header('Content-Type', 'application/octet-stream')
            with open(fpath, 'r') as f:
                s.write(f.read())
        elif os.path.exists(fpath) and os.path.isdir(fpath):
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Listing files in {fpath}:\n")
            for f in os.listdir(fpath):
                s.write(f"- {f}\n")            
        else:
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Oops, not found.")

        self.end_headers()

        self.wfile.write(s.getvalue().encode())


if __name__ == "__main__":
    PORT = 8000
    HANDLER = FileServerHandler
    with socketserver.ThreadingTCPServer(("0.0.0.0", PORT), HANDLER) as httpd:
        print("serving at port", PORT)
        httpd.serve_forever()

```

# GET Route :

As Mentioned Below, '```/```' will Return file in `/usr/src/app/files/`

<img src="https://i.imgur.com/9Kj8i21.png">

Flag is in `/usr/src/app/flag.txt`

# Base Path :

`BASEPATH="/usr/src/app"`

# FileServerHandler :

```python
class FileServerHandler(http.server.SimpleHTTPRequestHandler):
    server_version = "Fil3serv3r"

    def do_GET(self):
        self.send_response(-1337)
        self.send_header('Content-Length', -1337)

        s = StringIO()

        path = self.path.lstrip("/")
        counter = 0
        while ".." in path:
            path = path.replace("../", "")
            counter += 1
            if counter > 10:
                s.write(f"No")
                self.end_headers()
                self.wfile.write(s.getvalue().encode())
                return

        fpath = os.path.join(BASEPATH, "files", path)

        s.write(f"Welcome to @gehaxelt's file server.\n\n")
        if len(fpath) <= len(BASEPATH):
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Hm, this path is not within {BASEPATH}")
        elif os.path.exists(fpath) and os.path.isfile(fpath):
            self.send_header('Content-Type', 'application/octet-stream')
            with open(fpath, 'r') as f:
                s.write(f.read())
        elif os.path.exists(fpath) and os.path.isdir(fpath):
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Listing files in {fpath}:\n")
            for f in os.listdir(fpath):
                s.write(f"- {f}\n")            
        else:
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Oops, not found.")

        self.end_headers()

        self.wfile.write(s.getvalue().encode())

```

## path.lstrip :

```python
 path = self.path.lstrip("/")
```
lstrip will Just Remove the `/` comes Left side of the String

<img src="https://i.imgur.com/taF9OOl.png">

## While Loop :

```python
while ".." in path:
            path = path.replace("../", "")
            counter += 1
            if counter > 10:
                s.write(f"No")
                self.end_headers()
                self.wfile.write(s.getvalue().encode())
                return
```

The Above While Loop will Run If the Path Variable Have 2 Dots (`..`) 

## path.replace :

```python
path = path.replace("../", "")
```

`path.replace("../","")` will Search for `../` in Path Variable and Replace that with ``""``.

<img src="https://i.imgur.com/tfqb5eJ.png">

## path.join

```python
fpath = os.path.join(BASEPATH, "files", path)
```
We Have Control of the Path Variable!

## Exploit Idea :

After Some Trial and Error, I came Across this Interesting thing...

<img src="https://i.imgur.com/3lZrR4r.png">

When the Path Variable Starts with `/`, The Overall Path is Overwritten. Now We have Control of the Path. Let's Exploit

## If Else Condition Statements :

```python
        s.write(f"Welcome to @gehaxelt's file server.\n\n")
        if len(fpath) <= len(BASEPATH):
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Hm, this path is not within {BASEPATH}")
        elif os.path.exists(fpath) and os.path.isfile(fpath):
            self.send_header('Content-Type', 'application/octet-stream')
            with open(fpath, 'r') as f:
                s.write(f.read())
        elif os.path.exists(fpath) and os.path.isdir(fpath):
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Listing files in {fpath}:\n")
            for f in os.listdir(fpath):
                s.write(f"- {f}\n")            
        else:
            self.send_header('Content-Type', 'text/plain')
            s.write(f"Oops, not found.")

        self.end_headers()

        self.wfile.write(s.getvalue().encode())
```
These If Else Statements are Responsible for Sending Response. 

* If the Length of `fpath` is equal or lesser that `BASEPATH`, the Server will return, `Hm, this path is not within {BASEPATH}`


## Exploit : 

* Note: We Can Control Path Variable. 
* Our Input is Filtered Before Passed into 
<br>
`fpath = os.path.join(BASEPATH, "files", path)`

* To Exploit, Our Payload Want to Looks Like this, 
`fpath = os.path.join(BASEPATH, "files", "/usr/src/app/flag.txt")`  to Read the Flag. 

* Final Payload : `..//usr/src/app/flag.txt`

Let me Explain Why this Works: 

* After `path.lstrip`: 

<img src="https://i.imgur.com/pzetdOW.png">

* In While Loop, Replace Function is Called,

```python
path = path.replace("../", "")
```
After This Replace Function, 

<img src="https://i.imgur.com/UOfWycB.png">

* Then, Our Path Variable is Passed to `.join` 

And that Works :)

<img src="https://i.imgur.com/lMx4njb.png">

> FLAG: ENO{PYTH0Ns_0sp4thj01n_fun_l0l}
