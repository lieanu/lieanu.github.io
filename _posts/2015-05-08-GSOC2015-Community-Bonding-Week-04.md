---
layout: post
title: GSOC2015 Community Bonding Week 04
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 04

##Last week:

Automatic building rop chain for x86/x64/arm elf.

Using topological sorting to solve two issues: 

* Solved [ROP Register Dependency Resolution #21](https://github.com/binjitsu/binjitsu/issues/21), see `set_registers()` in rop.py module.
* Solved [Shellcode Register Dependency Resolution #22](https://github.com/binjitsu/binjitsu/issues/22), see `solve_register_dependencies()` in rop.py module.

Others:

* Do a survey on Amoco project [https://github.com/bdcht/amoco](https://github.com/bdcht/amoco).
* topological sorting see `__build_top_sort()` function.
* Rewirte X64/ARM ROP chain building methods, see `__build_x64()`, `__build_arm()`

##Next week:

* Need extract gadgets finding to a python Class.
* Try to using Amoco instead of BARF project.
* Merge to new rop module.
