---
layout: post
title: GSOC2015 Students coding Week 04
catagories: [gsoc2015]
tags: [gsoc2015, pwn]
---

> week sync 08

##Last week:

* Advance feature supported, see issue [#27](https://github.com/binjitsu/binjitsu/issues/27) .
* ARM ROP chain supported, see the example [armpwn exploit](https://github.com/lieanu/test_for_multiarch_rop/blob/master/arm/armpwn/pwn_armpwn_auto.py)
* Simplify the gadgets, when binary size large enough.
* Drop gadgets who's branch `>= 2`, except `call reg; xxx; ret` / `blx reg; xxx; pop{.*pc}` / `int 0x80; xxx; ret;` / `svc; xxx; ret`
* All in the [pull request #24](https://github.com/binjitsu/binjitsu/pull/24/commits)


##Next week:

* Optimizing and fix potential bugs.
* Add some doctests and pass the example doctests.
* [https://github.com/binjitsu/binjitsu/issues/27](https://github.com/binjitsu/binjitsu/issues/27)
