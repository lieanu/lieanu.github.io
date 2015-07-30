---
layout: post
title: GSOC2015 Students coding Week 10
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 14

##Last week:

* ReWritten all gadgets graph parts using networkx library.
* Using the algorithms in networkx.algorithms instead of the previous codes.
    * `networkx.topological_sort()` instead of `ROP.__build_top_sort()`
    * `networkx.all_shortest_paths()` instead of `ROP.__dfs()`.
* Filter all binaries as the `rop-tools` doing. regardless of its size. Important!!!
* `search_path()` return no more than 10 paths(shortest order), for performance.
* Using gadget's address as the node of graph, not the Gadget object, for performance.
* Update doctests and regular expression of filter.

It is much more faster now. We can finding gadgets and solving for `setRegisters()` within less than 10 seconds for most of the binaries, including the amoco's loading time which is about 2 seconds.

And classifier is the main bottleneck now.

##Next week:

* Fixing potential bugs.
* Aarch64 supported.
