---
layout: post
title: GSOC2015 Community Bonding Week 01
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 01

Last week:

* Read most of the barf's codes
    * Github: [https://github.com/programa-stic/barf-project](https://github.com/programa-stic/barf-project).
* Read the papers about the BARF project, REIL intermediate language, and Q: Exploit Hardening Made Easy

    * [BARF: A multiplatform open source Binary Analysis and Reverse engineering Framework](https://github.com/programa-stic/barf-project/blob/master/documentation/papers/barf.pdf)
    * [REIL: A platform-independent intermediate representation of disassembled code for static code analysis](http://static.googleusercontent.com/media/www.zynamics.com/en//downloads/csw09.pdf)
    * [Q: Exploit Hardening Made Easy](http://users.ece.cmu.edu/~ejschwar/papers/usenix11.pdf)

* Submited a pacth for BARF project
    * [https://github.com/programa-stic/barf-project/pull/11](https://github.com/programa-stic/barf-project/pull/11).
* Done a simple x86 ROP implementation.
* Coding for x64 now.

Next week:

* Do some ARM ROP examples
    * [https://github.com/binjitsu/examples/issues/2](https://github.com/binjitsu/examples/issues/2).
* Continue coding for x64 ROP.

There is  a problem for BARF project. While finding gadgets in large libraries, the efficiency is a problem. I compared three gadget tools(rop-tool, ROPgadget, BARFgadget):

`rop-tools`(written in `c`):

1229 gadgets found.

rop-tool gadget libc.so.6  **17.29s** user 0.01s system 100% cpu 17.289 total

`ROPgadget`:

Unique gadgets found: 21240

ROPgadget --binary libc.so.6  **72.30s** user 10.25s system 99% cpu 1:22.82 total

`BARFgadget`:

*           Find Stage :  **358.472s**
* Classification Stage :  854.280s
*   Verification Stage :  377.223s
*                Total : **1589.976s**


Open issus: [https://github.com/programa-stic/barf-project/issues/15](https://github.com/programa-stic/barf-project/issues/15).

Here is presentation for my classmates in tsinghua university NISL lab, it is in chinese, sorry for that.

[Automate ROP - 基于BARF的ROP链自动生成]({{site.baseurl}}/presentations/automate_rop.html)

