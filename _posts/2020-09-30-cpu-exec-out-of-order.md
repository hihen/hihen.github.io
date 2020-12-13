---
title: 证明CPU乱序执行的一种简单方式
author: 旺旺
date: 2020-09-30 22:07:00 +0800
categories: [Hardware, CPU]
tags: [Hardware, CPU, ExecOutOfOrder]
---

上一篇文章分享了volatile的实现原理，其中提到了CPU的乱序执行，这篇文章来分享一下CPU乱序执行的一种最简单证明。

将会从以下几个方面去作出分享:
* 如何证明CPU乱序执行
* 为什么要开多线程去证明

### 如何证明CPU乱序执行

通过一段很简单的Java代码就能证明CPU存在乱序执行，现在我把这段代码贴在下面。

```java
public class CpuExecOutOfOrderTest {
    
    private static int s1, s2, t1, t2;
    
    public static void main(String[] args) throws InterruptedException {
        for (int i = 1; ; i++) {
            s1 = 0;
            s2 = 0;
            t1 = 0;
            t2 = 0;
            Thread first = new Thread(() -> {
                t1 = 1;
                s1 = t2;
            });
            Thread second = new Thread(() -> {
                t2 = 1;
                s2 = t1;
            });
            first.start();
            second.start();
            first.join();
            second.join();
            if (s1 == 0 && s2 == 0) {
                System.out.println("when " + i + "th exec, s1 = 0 and s2 = 0.");
                break;
            }
        }
    }
}
```

程序中定义了四个int变量s1, s2, t1, t2，然后用两个线程来分别对他们进行赋值操作。外面有一层for循环包裹着，无休止的执行，直到出现CPU乱序执行的结果，程序终止。

从代码中可以看到，for循环终止的条件是's1 == 0 && s2 == 0'。也就是说，当s1和s2同时为0时，CPU执行出现了乱序。是不是一时之间想不明白为啥要这么证明？没关系，我们来做一波假设推理。

假设CPU是严格按照代码顺序来执行，那么两个线程整体的执行顺序将会有以下几种排列组合情况(为了防止眼花，将4条语句分别编排为a,b,A,B。排列组合的前提是a一定在b前面，A一定在B前面)：

```console
a: t1 = 1;
b: s1 = t2;
A: t2 = 1;
B: s2 = t1;

result: s1=0,s2=1
```

```console
a: t1 = 1;
A: t2 = 1;
b: s1 = t2;
B: s2 = t1;

result: s1=1,s2=1
```

```console
a: t1 = 1;
A: t2 = 1;
B: s2 = t1;
b: s1 = t2;

result: s1=1,s2=1
```

```console
A: t2 = 1;
a: t1 = 1;
B: s2 = t1;
b: s1 = t2;

result: s1=1,s2=1
```

```console
A: t2 = 1;
B: s2 = t1;
a: t1 = 1;
b: s1 = t2;

result: s1=1,s2=0
```

```console
A: t2 = 1;
a: t1 = 1;
b: s1 = t2;
B: s2 = t1;

result: s1=1,s2=1
```

上面的分析，是CPU按序执行时会出现的所有组合情况，执行结果都没有出现s1和s2同时为0的情况。反过来，如果CPU会出现乱序，则会出现s1和s2同时为0的情况。

要证明一个东西是好的，那需要证明所有情况下它都是好的。但是要证明一个东西是坏的，则只需要证明一种情况下它是坏的就可以。下面是我举出的一组例子，证明了s1和s2同时为0是CPU乱序执行的结果：

```console
b: s1 = t2;
B: s2 = t1;
a: t1 = 1;
A: t2 = 1;

result: s1=0,s2=0
```

除了理论推理，再附上我的实际代码运行结果吧。请看图！

![cpu-exec-out-of-order-test](http://www.giver.vip/article_image/cpu-exec-out-of-order-test.png)

### 为什么要开多线程去证明

在知道怎么证明CPU存在乱序之后，不知道大家有没有疑问。反正我是有这样一个疑问: 为什么要开两个线程去证明？这又不是验证并发情况下的数据不一致性，有必要开两个线程吗？

答案是: 非常有必要。这个问题的背后，又回到了CPU乱序执行的产生原因。为啥存在乱序执行，一段指令重新组合，能让CPU能更加高效的执行，不要让CPU有偷懒的时间呗。
这个重新组合有个非常重要(此处省略一万个感叹号)的前提，那就是在单线程的环境中，不管CPU怎么乱搞，最终执行结果要和按序执行的结果完全一致。

现在应该没有人不明白了吧（如果有，那就再重新读两边哈哈），如果只用一个线程去执行四条语句，那CPU就不敢乱搞了，它得保证最终执行结果都是同一个。

对于CPU这个规矩，有两个专业概念的，它们分别是: happens-before，as-if-serial，推荐大家去了解一波吧，去oracle官网了解哦。好了，干货就写到这吧。

每年中秋前一天，是小瑾同学的生日。本该万般呵护，相聚甚欢的日子，却因为距离，因为学业，整个生日都是陪着冷冰冰的实验仪器一起渡过。小瑾生日快乐！委屈你了，期待我们愉快开心的国庆假期，补偿你应有的快乐～

