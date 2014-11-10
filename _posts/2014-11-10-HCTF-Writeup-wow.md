---
layout: post
title: HCTF2014 wow writeup
catagories: [ctf]
tags: [ctf, writeup, hctf, xctf]
---
> wow的这一题的binary同时也是“真的能做吗”的binary.


IDA可以看到关键问部分在`check()`函数里，wow的位置存了22个字符串，每个字符串的位置大小是65字节，`GetSentence()`这个函数做的事情，就是把字符串里的非字母的东西的过滤掉，然后不足65字节的全部补0.

`verify`里保存的是要与之比较的值。

`verify`验证的上一条语句做的事情是：输入的前22个字节与每个字符串的前22字节，分别相
乘再相加。因此最终的结果就是一个22×22的矩阵A与一个22x1的矩阵B相乘，得到一个22x1的矩阵C.
其中,矩阵C和矩阵A已经，求矩阵B.那么`B = A.I * C`.这样即可算出矩阵B。

```python
#!/usr/bin/env python2
# -*-coding: utf-8 -*-
import struct
import numpy

verify = 'ca730300df1b0300f774030006940300c4990300dc4a0300088c0300888b0300608a030068b5020071240300ea7d0300976f0300e47803000687030010900200234c0300f88e0300298e03005e920300fcb502004e580200'.decode("hex")

#把verify的值对应放到result[]里，即矩阵C
result = []
for i in range(22):
    result.append([struct.unpack("i", verify[(4*i):4*(i+1)])[0]])
print result

a=["ThelightTokeepinmindtheholylight",
"Timeismoneymyfriend\x00\x00\x00\x00\x00",
"WelcometotheaugerRuiMa\x00\x00\x00\x00\x00",
"Areyouheretoplayforthehorde\x00\x00\x00\x00\x00",
"ToarmsyeroustaboutsWevegotcompany\x00\x00\x00\x00\x00",
"Ahhwelcometomyparlor\x00\x00\x00\x00\x00\x00",
"Slaytheminthemastersname\x00\x00\x00\x00\x00",
"YesrunItmakesthebloodpumpfaster\x00\x00\x00\x00\x00",
"Shhhitwillallbeoversoon\x00\x00\x00\x00\x00",
"Kneelbeforemeworm\x00\x00\x00\x00\x00\x00\x00\x00",
"Runwhileyoustillcan\x00\x00\x00\x00\x00\x00",
"RisemysoldiersRiseandfightoncemore",
"LifeismeaningleshThatwearetrulytested",
"BowtothemightoftheHighlord\x00\x00\x00\x00\x00",
"ThefirstkillgoestomeAnyonecaretowager",
"Itisasitshouldbe\x00\x00\x00\x00\x00\x00\x00",
"Thedarkvoidawaitsyou\x00\x00\x00\x00\x00\x00",
"InordertomoregloryofMichaelessienray",
"Rememberthesunthewellofshame",
"Maythewindguideyourroad\x00\x00\x00\x00\x00",
"StrengthandHonour\x00\x00\x00\x00\x00\x00\x00\x00\x00",
"Bloodandthunder\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"]


#把a的值对应放到b里，形成矩阵A
b = []
for i in range(22):
    mid = []
    for one in range(22):
        mid.append(struct.unpack("b", a[i][one])[0])
    b.append(mid)
print b

#这样就提取了矩阵b（22x22），和结果r（22x1)
mb= numpy.matrix(b)
mr = numpy.matrix(result)

print mb.I
#mr.I为mr的逆矩阵
mi = numpy.dot(mb.I, mr)
print mi
```

得到结果，并处理：

```python
mi = [[ 104.],
 [  99.],
 [ 116.],
 [ 102.],
 [ 123.],
 [  76.],
 [  74.],
 [  95.],
 [ 121.],
 [  54.],
 [  99.],
 [ 100.],
 [  99.],
 [  95.],
 [ 113.],
 [ 119.],
 [ 101.],
 [ 101.],
 [ 114.],
 [ 116.],
 [  33.],
 [ 125.]]

for ii in mi:
    for j in ii:
        print chr(int(j)),
```

获取flag:`hctf{LJ_y6cdc_qweert!}`


