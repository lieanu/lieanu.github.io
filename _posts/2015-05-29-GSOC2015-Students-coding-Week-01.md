---
layout: post
title: GSOC2015 Students coding Week 01
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 05

##Last week:

* Rewrite the module using `amoco` project.
    * Extract gadgets using capstone, we will get the gadget address, insns, raw bytes.
    * Classify gadgets using amoco.
        * Using amoco project to symbolic execution gadgets, return a mapper.
        * Caculate the sp move, know the gadget move.
        * Tidy the regs relationships, got gadget regs.
        * Discard `mem[xxx] <- zzz` gadgets.
    * Solver gadgets using amoco.
        * Symbolic execution gadget, return a mapper.
        * Get the symbolic expression of every output regs.
            * Example: `pop rdi; ret`. we need `rdi == 0xbeefdead`.
            * Get `rdi <- { | [0:64]->M64(rsp) | }` from mapper.
            * Convert this expression to z3 format. `expression == 0xbeefdead` 
        * Using z3 solver, add the condition above, and solve it.
        * Get the content in `mem[rsp]`.
* Extract gadget finder/classify/solver process respectively to a python class.
* Merge the module to binjitsu master branch and submit a PR.
    * [https://github.com/binjitsu/binjitsu/pull/24](https://github.com/binjitsu/binjitsu/pull/24)


##Next week:

* Do something as zach suggested in PR comments.
* doctests/examples for all the methods in this module.
* Topological sorting has some bugs needed to be fixed.
