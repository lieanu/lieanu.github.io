---
layout: post
title: GSOC2015 Students coding Week 09
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 13

##Last week:

* Single process optimization for `load_gadgets() and build_graph()`
* Multi Process supporting for `GadgetFinder.load_gadgets()`
* Multi Process supporting for `ROP.build_graph()`

Example for `libc.so.6`, which size larger than 200Kb.

```
 lieanu@ARCH $ time python -c 'from pwn import *; context.clear(arch="amd64"); rop=ROP("/usr/lib/libc.so.6")' 
[*] '/usr/lib/libc.so.6'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
python -c   44.18s user 3.04s system 301% cpu 15.655 total       
```

Example for `xmms2`, < 200Kb

```
 lieanu@ARCH $ ls -alh /bin/xmms2 
-rwxr-xr-x 1 root root 133K Jun  4 18:27 /bin/xmms2
 lieanu@ARCH $ time python -c 'from pwn import *; context.clear(arch="amd64"); rop=ROP("/bin/xmms2")'
[*] '/bin/xmms2'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE
python -c   86.14s user 1.05s system 305% cpu 28.545 total       
```

* bottlenecks:
    1. All graph operation, such as: top sort and dfs.
    2. Classify when finding gadgets.

##Next week:

* Optimization for graph operation.
* Fixing potential bugs.
