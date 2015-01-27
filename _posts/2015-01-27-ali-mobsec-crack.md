---
layout: post
title: 阿里移动安全挑战赛小记
catagories: [ctf]
tags: [android, ali, dex]
---

> 从零开始学习Android逆向/混淆/反调试

##0x00 前言

话说阿里要搞移动安全挑战赛，持续3天，正好学期末事情比较少，对Android也是从来没接触过，
那就通过这次比赛去学习一下吧。

> 虽说是单人赛，但期间还是跟同学有过不少讨论交流，整个过程收获也比较大。

##0x01 Crackme 1

没搞过Android，拿到APK以后，惯例file一下，再果断unzip之，有个叫`classes.dex`的文件，
dex文件是Dalvik虚拟机的字节码文件，找个工具反编译它吧。随便找了一个叫做`jadx`的工具。
扔进去，看了看代码，就是对`logo.png`这个文件来回的截取，写个脚本搞它（脚本比较乱，没整理）：

```python
#!/usr/bin/env python2
# -*- coding: utf-8 -*-

#fd = open("logo.png", "r")
#a = fd.read()
#a =  a.encode("hex")
##b = a[89473*2:89473*2+768*2]
#b = a[91265*2:91265*2+18*2]
##b = a[89473:89473+768]
##b = a[89473:]
#b=b.decode("hex")
#print len(b)
#print b
#print b.decode("hex")

#
table = u"一乙二十丁厂七卜人入八九几儿了力乃刀又三于干亏士工土才寸下大丈与万上小口巾山千乞川亿个勺久凡及夕丸么广亡门义之尸弓己已子卫也女飞刃习叉马乡丰王井开夫天无元专云扎艺木五支厅不太犬 区历尤友匹车巨牙屯比互切瓦止少日中冈贝内水见午牛手毛气升长仁什片仆化仇币仍仅斤爪反介父从今凶分乏公仓月氏勿欠风丹匀乌凤勾文六方火为斗忆订计户认心尺引丑巴孔队办以允予劝双书幻玉刊 示末未击打巧正扑扒功扔去甘世古节本术可丙左厉右石布龙平灭轧东卡北占业旧帅归且旦目叶甲申叮电号田由史只央兄叼叫另叨叹四生失禾丘付仗代仙们仪白仔他斥瓜乎丛令用甩印乐"
pw = u"义弓么丸广之"

for one in pw:
    r = table.find(one)
    print chr(r),
```

##0x02 Crackme 2

拿到APK后，再unzip它，同样的把`classes.dex`扔进去看看，没啥东西，不过有个`JNI`的调用，
调了native代码`libcrackme.so`。`JNI`是个啥，简单补了一下，就是Java代码与C互相调用的一个机制。

从这里看，那么事情应该主要在`libcrackme.so`，这个文件里。掏出IDA，看到`Java_com_yaotong_crackme_MainActivity_securityCheck`这么个函数，`while`循环之前，有个本地变量，其指的值是`wojiushidaan`，靠，这么直接。直接把这个字符串，扔进去，果断不对。那么这个值应该还是被其它函数改动过。

到这里在想，把程序跑起来，那么它内存里的值应该是改过的，那`while`循环之前，下个断点，直接看内存的值，应该就可以。但非常郁闷的是，程序做了反调试。

这时候补了一下`adb`相关的命令和`gdbserver`相关调试方法，`gdbserver`forward出来，还是用`gdb`连的它，何不直接在手机上运行`gdb`，找了一个叫`gdb-static`的工具，解出来有两个文件`gdb/gdbserver`。

没写过Android代码，也没搞过这一块的反调试，打开脑洞猜吧。yhy写了几个对应的解密的函数，对一堆可疑的字符串解密后，是一些诸如`dlsym/getpid`之类的libc函数。这不应该是反调试的地方啊，没有对应的敏感函数。简单故意改错几个字符串，然后再重打包。

没搞过重打包，又要恶补一下了。这里找到了`apktools`，才发现`unzip`的方法太暴力，不能重打包回来。替换了改过的so文件后，重打包。然后`adb install ***.apk`，失败了。证书不对，还要重签名，搜了一下重签名的命令，还没签好。找一找有没有更方便的重签名工具吧。这里最终用了它[https://github.com/appium/sign.git](https://github.com/appium/sign.git),还是很方便的。

安装成功，运行一下，程序跑不起来，输入的textedit都不弹出来。应该是改失败了。再猜。

继续分析代码后，发现`JNI_OnLoad`里有个`v5 = dword_62B4(handle, 0, sub_16A4, 0);`不知道干啥用的。试着把它NOP掉，再重打包吧。好吧，猜对，可以调试了。在`0x000012A4 + offset`下个断点，`ni`一下，`x/10s $r2`读到flag。

这里上面这个`offset`是指这个so被映射到内存中的基址，这个地址怎么看？`cat /proc/xxx/maps`就可以看到。

##0x03 Crackme 3

简单分析一下，`classes.dex`里啥都没有，`lib`里，有3个so,有一个是正经的`elf32`文件，另一个竟然是`jar`伪装的，还是一个是纯`data`的，暂时不知道干啥用的。把`jar`伪装的那个拿出来，反编译一下，发现函数竟然都是空的。判断应该是加壳了，虽然对加壳的方法一窍不通。但是先这么判断吧。

试着调试，会告诉我`ptrace: no permission`之类的提示，再google大法，判断应该是程序自己调了`ptrace`来ptrace自己，然后gdb因为也需要`ptrace`来实现调试的功能，连续两次被`ptrace`搞，就对弹这么个问题。google大法告诉，这是实现反调试的一种方法，梆梆加固有种反调试的方法，就跟此类似，用三个进程互相`ptrace`。

先试着将所有调ptrace给NOP掉，或者改plt表，换成类似`sleep/puts`之类的函数。重打包签名后，
果断还是不行。不过CHO告诉我，他把ptrace给NOP掉是可以的，可能是我的姿势不对。
不过可以试着`attach`到线程上，因为是共享内存空间的，导出的内存应该是一样的。

cd到`/proc/xxx/tasks`下面，`ls`一下，就在这时，程序退出了。反复试了几个依然如此。反调试搞这么叼，线程号都不给看。
那就猜，看了下，线程号基本上都是顺着这个程序的进程号依次往下排的。那就`attach`到`pid+1/2/3/4/5`。
这么多，总有一个是它的线程吧。然后用`gcore`导出当前内存。

下面是从内存中搜`dex`文件的事情了，简单写个脚本处理一下：

```python
#!/usr/bin/env python2
import sys
import os
import struct

MAGIC = "\x64\x65\x78\x0a\x30\x33\x35" #dex.035

def parse(filePath):
    if not os.path.isfile(filePath):
        print "Not a File"
        return 0
    offset = 0
    
    outputDir = "dump_" + os.path.basename(filePath) + "/"
    if not os.path.exists(outputDir):
        os.mkdir(outputDir)

    fd = open(filePath, "r")
    data = fd.read()
    while True:
        pos = data[offset:].find(MAGIC)
        if pos == -1:
            break
        dexsize = struct.unpack("I", data[offset + pos + 32 : offset + pos + 36 ])[0]
        
        outputName = hex(offset + pos) + ".odex"
        ff = open(outputDir + outputName, "wb")
        ff.write(data[offset + pos : offset + pos + dexsize])
        ff.close()
        #print pos, dexsize

        offset = offset + pos + dexsize

    return 1

if __name__ == "__main__":
    if len(sys.argv) < 2 :
        sys.exit()
    parse(sys.argv[1])
```

在取出来的`odex`文件里，`grep`一下有没有`crack`相关的敏感字符，还真有一个dex是有的。
拿出来用各种反编译工具都试了下，报错了。错在何处？看到了，其实也不懂。

因为不懂`dex`的格式，再google大法恶补一下吧。[看看官方的文档](https://source.android.com/devices/tech/dalvik/dex-format.html)

一番学习之后，再用`010editor`解析一下吧，更方便看。就在这时，`010editor`告诉我试用期到期，请购买。
本屌丝没钱，只有试着破解它，扔到`ida`发现，符号表都在，改一个条件跳转为无条件跳转，秒破了。不会吧，
一点保护措施都没有。

继续用`010editor`分析，发现到`StubRuntimeException`的`annotation_off`时就出错了，它的值是32,试着将它改成0,再解析一下。这时可能看到`data`段相应的东西了。往下继续看，发现到`class_data->direct_method->method->code_off`时，又跪了。

`code_off`竟然是`0xffffe094`，这不是负数吗？按照官方的文档解释，这是要从文件头往前找的节奏。
原来`code`段的东西在内存中另外一个地方，这也能解释了为何`lib`文件夹下，会有一个纯`data`的`so`文件。

怎么办？手动一个一个改？但离比赛仅有半个小时了，肯定来不及了。默默的等到比赛结束，CHO告诉我，可以改smali的源码来解析。

我将这个dex在内存中之前的所有memory,给接到这个dex的后面。那么偏移，就应该是`dexsize(0x3004) + memorysize`，这么大。

以下是改动的smali源码的部分。

```diff
diff --git a/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/BaseDexReader.java b/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/BaseDexReader.java
index 96645b8..b137bc6 100644
--- a/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/BaseDexReader.java
+++ b/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/BaseDexReader.java
@@ -130,6 +130,9 @@ public class BaseDexReader<T extends BaseDexBuffer> {
         }
 
         offset = end;
+       if ((result & 0xff000000) == 0xff000000) {
+           result = result + 0x3fe64;
+       }
         return result;
     }
 
diff --git a/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/DexBackedMethod.java b/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/DexBackedMethod.java
index f26b7e1..dc7b285 100644
--- a/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/DexBackedMethod.java
+++ b/dexlib2/src/main/java/org/jf/dexlib2/dexbacked/DexBackedMethod.java
@@ -97,7 +97,8 @@ public class DexBackedMethod extends BaseMethodReference implements Method {
         int methodIndexDiff = reader.readLargeUleb128();
         this.methodIndex = methodIndexDiff + previousMethodIndex;
         this.accessFlags = reader.readSmallUleb128();
-        this.codeOffset = reader.readSmallUleb128();
+        //this.codeOffset = reader.readSmallUleb128();
+        this.codeOffset = reader.readLargeUleb128();
 
         this.methodAnnotationSetOffset = methodAnnotationIterator.seekTo(methodIndex);
         this.parameterAnnotationSetListOffset = paramaterAnnotationIterator.seekTo(methodIndex);
```

我这里的偏移是`0x3fe64`。再次用`baksmali`进行解析，出了一些错，百思不得其解，然后重新dump内存，dump内存之前，先在程序的文件框里随便输入点什么东西，再点下按钮。

再按上面所说的方法，把dex取出来，接上前面的一段内存，当然，相应的偏移肯定不一样了。
再用`baksmali`解析，几乎成功了，只有一个地方报错，`StubRuntimeException`的`annotation_off`，把它的值改成0.
非常另人激动，smali的代码，经过千辛万苦终于搞出来了。

借助google大法，现场再恶补一下`smali`的语法，经过几个小时的学习后，自信满满的地看smali代码，被`b.run()`里的内容彻底搞晕了，明显学的还不够嘛。

有没有把smali转成java的工具，按道理来说，是能实现的。google大法这次不给力了，它介绍的各种`smali2java`的工具，都不能工作。把smali代码扔给小伙伴们看看吧。

yhy说`baksmali/smali.jar`就能实现从`smali`到`dex`的互相转换啊。真是空守宝山而不自知，试着用`smali.jar`将`smali`
代码转成`dex`文件，再用`dex2jar`工具或扔到`jadx-gui`里看，有些函数没反编译出来，这是什么问题？
看了看smali的代码，发现里面有这么个函数`testdex2jarcrash`，一看名字，就知道它不会干什么好事。果断把它注释掉。再搞一遍，终于出java代码了。

这里感慨`testdex2jarcrash`函数的牛逼啊，没几个代码，怎么能实现`anti-decompile`。somebody看了看，说问题出现`.end param`这个地方，这个语法是不对的，所以解不出来。也就是只要把这一句，注释掉就可以了。如下：

```ruby
.method public testdex2jarcrash(Ljava/lang/String;Ljava/lang/String;)V
    .registers 3
    .param p1, "test1"    # Ljava/lang/String;
    .param p2, "test2"    # Ljava/lang/String;
        .annotation build Lcom/system/TestA;
        .end annotation
    #.end param   #这里这句话注释掉。

    return-void
.end method
```

好，现在终于有了java的代码，看雪上各路大神，也放出了自己的writeup。没关系，我继续自己的。

`b.run()`终于是人类能看的代码了，分析一下，应该在这个地方：

```java
char[] toCharArray = a.toCharArray();
value = a.substring(0, 2).hashCode();
if (value > 3904) {
    //...
} else if (value == 3618 && toCharArray[0] + toCharArray[1] == 168) {
    do {
        byte[] bytes = (((f) e.class.getAnnotation(f.class)).a() + ((f) a.class.getAnnotation(f.class)).a()).getBytes();
        //... //做了一些比较的事情
    } while (bArr == null);
```

`getAnnotation`这个函数，查一查就知道，取的相应的值，这里分别是`7e`和`1p`。

`hashCode`是怎么实现的，万能的`stackoverflow`告诉我，是这样的：

```java
public int hashCode() {
    int hashCode = 1;
    Iterator i = iterator();
    while (i.hasNext()) {
        Object obj = i.next();
        hashCode = 31*hashCode + (obj==null ? 0 : obj.hashCode());
    }
    return hashCode;
}
```

这样事情就简单了，也就是两个字符`"ab"`要同时满足`(ord(a)*31 + ord(b)) == 3618 && (ord(a) + ord(b)) == 168`，跑一下，得到`s5`。

那么最终的答案，应该是`s57e1p`的moss码了，但是当然不是标准的moss码，类`e`里有个`Map`，按照他的方式编码就是答案。

##0x04 小结

比赛早已结束，我第三题才算做完，已经没法再看第四题。被虐很惨，感触很深。

在分析apk时，想用一些动态分析的工具，看了`0ops`在`alictf`时用`InDroid`来秒题，找了一下，没有文档，也不会用。
再仔细一看，`InDroid`的作者，不就是这次比赛的第一名么。ORZ，果断差距没法比。

再就是脑洞完全不够，第二题CHO瞬秒了，竟不知有反调试。直接改二进制，把flag打出来的。果然正面硬上，往往难度都是要大滴。

通过这次比赛，也算对Android的一些工具/技巧，有了一个简单的了解。还是感觉学习很多，比如调试方法/dex格式/smali语法/各类工具的收集与使用。

感谢阿里移动安全挑战赛，让本来对移动安全不感冒的我，也升起了极大的兴趣。
