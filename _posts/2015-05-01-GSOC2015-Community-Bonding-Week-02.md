---
layout: post
title: GSOC2015 Community Bonding Week 02
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 02

##Last week:

* Finish a simple implementation, support x86 and x64 now.
    * [https://github.com/lieanu/binjitsu/blob/multirop/pwnlib/rop.py](https://github.com/lieanu/binjitsu/blob/multirop/pwnlib/rop.py)

###How?

Phase 01: Extract gadgets.

* Extract gadgets using capstone.
* Filter gadgets when ELF file size more than 100KB now.
* Cache gadgets in a tmp file, as old rop module do.
* Convert gadgets into REIL instructions using BARF
* Classify and verify gadgets using BARF. **Here is the bottleneck of performance**

Phase 02: Search gadgets for specific regs

* Build a gadgets graph. gadget as vertex, reg as edge.
* Find out gadget with speicific regs using DFS algorithm.
* Sort all gadgets base on the length of gadget. gadgets[0] will be used.

Phase 03: Building ROP chain.

* Using `call()` function to add a func and arguments
* Decide to find which gadgets base on the number of auguments.
* Solve the gadget to decide the value on the stack. FOR EXAMPLE: `system("/bin/sh")`, using `pop rdi; ret`.
    * One argument, so `pop rdi; ret` will be used, Search it out as Phase02.
    * Add REIL instructions of "pop rdi; ret" to code_analyzer module of BARF project.
    * Using `rdi = analyzer.get_register_expr("rdi", mode="post")` to get the `rdi` post value.
    * Using `analyzer.set_postcondition(rdi == address_of_binsh)` to set a condition.
    * Using `stack.append(analyzer.get_memory_expr(rsp_pre + i*8, 8))` to simulate a **Stack**.
    * Using z3 to check it. And we got the value on stack before instructions execute.
* We know all things now: gadget's address, stack, function address, so it's easy to build x64 rop chain. 

##Next week:

* Go on optimizing the performance of this module. 
    * After extract gadgets with captone and cache gadgets in a tmp file, the performance is much better now.
    * The classify pass of BARF is a bottleneck now. 170 instructions cost about 50 seconds.
    * I just wanna to know the destination and source regs of one gadgets, maybe the classify pass is unnecessary. Find a way to replace it.
* Functions backward compatibility.
    * Do like old rop module do.
* Testing and Bugs fix. Maybe a lot of bugs haven't be founded.
* Do some ARM ROP examples.
    * [https://github.com/binjitsu/examples/issues/2](https://github.com/binjitsu/examples/issues/2).

