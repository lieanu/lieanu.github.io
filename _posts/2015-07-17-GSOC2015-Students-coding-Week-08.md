---
layout: post
title: GSOC2015 Students coding Week 08
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 12

##Last week:

* Update the doctests for ROP module.
    * The bug when using more than one cpu archs at the same time.
    * [https://gist.github.com/lieanu/f65788ff947a04d50aa0](https://gist.github.com/lieanu/f65788ff947a04d50aa0#)
    * [https://github.com/bdcht/amoco/issues/21](https://github.com/bdcht/amoco/issues/21)
* Update the doctests for gadgetfinder module.
* Using LocalContext to get the binary arch and bits.
* Start coding for Aarch64 supported.
* Try to do some code optimization.

    ```
           220462    0.743    0.000    0.760    0.000 :0(isinstance)
           102891    0.413    0.000    0.413    0.000 :0(match)
    116430/115895    0.347    0.000    0.363    0.000 :0(len)
             1119    0.243    0.000    0.487    0.000 :0(filter)
            80874    0.243    0.000    0.243    0.000 glob.py:82(<lambda>)
            11226    0.117    0.000    0.117    0.000 :0(map)
      12488/11920    0.047    0.000    0.050    0.000 :0(hash)
    ```

* Fix some bugs in rop module.

##Next week:

* Coding for Aarch64.
* Optimizing and fix potential bugs.
* Add some doctests and pass the example doctests.
