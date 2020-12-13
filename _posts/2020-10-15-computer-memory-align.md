---
title: 计算机系统为什么要内存对齐？
author: 旺旺
date: 2020-12-13 18:08:00 +0800
categories: [Hardware, Memory]
tags: [Hardware, Memory, CPU]
---

有时候会遇到同学们在问：“我打印出来的Java对象大小为什么总是24bytes，32bytes，40bytes，为什么不能是25bytes，26bytes，27bytes呢？除此之外，我发现它们都是8的倍数唉，好奇怪！“。为了让这些同学不再疑惑，于是我决定写下这篇文章。
本文将会从以下几个方面去分享:

* Java中为什么会有这个现象
* 如何使用C语言证明内存对齐
* 计算机为什么要使用内存对齐

### Java中为什么会有这个现象

开局一张图，这是Oracle Java8的官方文档解释，茫茫英文文海里翻出来的不容易，同学们细品。

![jvm-official-doc-memory-align](http://www.giver.vip/article_image/jvm-official-doc-memory-align.png)

官网想告诉我们的呢，其实就是内存中的Class文件是以字节流的形式存储的。所有16bit、32bit和64bit的数据分别都是通过读取2bytes、4bytes和8bytes的内存空间来构造存储的。多字节数据项总是以高位顺序存储，高字节优先。

好了，这段话其实已经足够解释同学们的疑惑了。Java中最大的数据类型long和double都是占8bytes，所以以最大的8bytes为单位来进行内存分配。官网文档真香！

另外，怎么打印出Java对象的大小，本文就不讲啦，可以借助Instrument的代理机制来实现，同学们可自行研究。

### 如何使用C语言证明内存对齐

有木有同学疑惑，为什么Java中出现的现象，要用C语言来证明呢？先做一波解释吧，因为JVM是用C写的，底层运行的就是C。对，就是这么简单。

先看一下本次的实验代码吧。

```console
#include<stdio.h>

int main(void)
{
	struct S{
		int x;
		char y;
		int z;
	};
	struct S s;
	printf("Address of x: %p\n", &s.x);
	printf("Address of y: %p\n", &s.y);
	printf("Address of z: %p\n", &s.z);
	
	return 0;
}
```

在C里边，一个int占4个字节，一个char占1个字节。同学们脑补的内存分配图会是怎样子的，是不是下面这种？

![memory-picture-false](http://www.giver.vip/article_image/memory-picture-false.png)

真实的内存环境中，是不是这样分配的呢？我们来看一下实验数据就知道啦。

![exam-result](http://www.giver.vip/article_image/exam-result.png)

上图中已经将x，y，z三个变量的内存地址分别打印出来了，计算他们之间的差值，就能知道x，y，z分别占了多少内存了。哦对了，这个地址是16进制表示，不是人类看的东西，我们来把它转一下10进制吧。

![10-jinzhi-result](http://www.giver.vip/article_image/10-jinzhi-result.png)

由于三个地址的区别在于后三位，所以我只对后三位做了转换。

从转换结果可得知，x的地址是2972，y的地址是2976，z的地址是2980。也就是说，x占了4bytes，y占了4bytes，z毫无疑问也是4bytes。相信最让同学们不解的就是，y是char类型为什么要占4bytes，这就是计算机设计思想里很有名的内存对齐了，现代的所有处理器设计中，基本都采用了这一原则（不用怕不是个傻子）。

### 计算机为什么要使用内存对齐

实际只需要占用一个byte的char类型，计算机却给它分配了4bytes。设计这计算机的人是不是傻了？哈哈哈，傻是不可能的，人家聪明着呢。

现代的计算机系统对基本数据类型的合法地址都做了限制，要求某种类型对象的地址必须是某个值（通常是2，4，8）的倍数。这种内存对齐限制简化了处理器和内存接口的设计。给同学们举个例子，假设一个处理器要从内存中读取8bytes的数据A，则A地址必须是8的倍数，这样就可以用一个内存操作来读或者写这个A了。否则，我们可能需要执行两次内存访问，因为A可能被放在了两个8bytes的内存块中。

到这里同学们应该都知道了，计算机的内存对齐这个设计采用的就是空间换时间的思想。毕竟CPU兔子般的速度，是无法忍受内存那乌龟般速度的，内存自身有自知之明，只能做出牺牲来尽量跟上CPU的步伐了。这个设计是真的香！还有，C语言也是真的香！

好的，这篇文章就分享完了，还有什么疑问可以邮箱私信我，hihenzhang@gmail.com。

写这篇文章，一个下午和晚上又没了。本来就很短暂的周末，几乎没有任何时间陪伴小瑾了，实在对不起，下周末咱不写了！！