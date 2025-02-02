---
layout:     post
title:      python--GIL锁
subtitle:   
date:       2019-04-08
author:     P
header-img: img/post-bg-ioses.jpg
catalog: true
tags:
    - python
---
GIL锁

**本节目录**

- [一 介绍](#part_1)
- [二 GIL介绍](#part_2)
- [三 GIL与Lock](#part_3)
- [四 GIL与多线程](#part_4)
- [五 多线程性能测试](#part_5)
- 



### 一 背景知识

```
'''
定义：
In CPython, the global interpreter lock, or GIL, is a mutex that prevents multiple 
native threads from executing Python bytecodes at once. This lock is necessary mainly 
because CPythons memory management is not thread-safe. (However, since the GIL 
exists, other features have grown to depend on the guarantees that it enforces.)
'''
结论：在Cpython解释器中，同一个进程下开启的多线程，同一时刻只能有一个线程执行，无法利用多核优势
```

　　首先需要明确的一点是`GIL`并不是Python的特性，它是在实现Python解析器(CPython)时所引入的一个概念。就好比C++是一套语言（语法）标准，但是可以用不同的编译器来编译成可执行代码。有名的编译器例如GCC，INTEL C++，Visual C++等。Python也一样，同样一段代码可以通过CPython，PyPy，Psyco等不同的Python执行环境来执行。像其中的JPython就没有GIL。然而因为CPython是大部分环境下默认的Python执行环境。所以在很多人的概念里CPython就是Python，也就想当然的把`GIL`归结为Python语言的缺陷。所以这里要先明确一点：GIL并不是Python的特性，Python完全可以不依赖于GIL

[　　这篇文章透彻的剖析了GIL对python多线程的影响，强烈推荐看一下：http://www.dabeaz.com/python/UnderstandingGIL.pdf ](http://www.dabeaz.com/python/UnderstandingGIL.pdf)

### 二 GIL介绍

　　GIL本质就是一把互斥锁，既然是互斥锁，所有互斥锁的本质都一样，都是将并发运行变成串行，以此来控制同一时间内共享数据只能被一个任务所修改，进而保证数据安全。

　　可以肯定的一点是：保护不同的数据的安全，就应该加不同的锁。

　　要想了解GIL，首先确定一点：每次执行python程序，都会产生一个独立的进程。例如python test.py，python aaa.py，python bbb.py会产生3个不同的python进程

```
'''
#验证python test.py只会产生一个进程
#test.py内容
import os,time
print(os.getpid())
time.sleep(1000)
'''
python3 test.py 
#在windows下
tasklist |findstr python
#在linux下
ps aux |grep python
```

在一个python的进程内，不仅有test.py的主线程或者由该主线程开启的其他线程，还有解释器开启的垃圾回收等解释器级别的线程，总之，所有线程都运行在这一个进程内，毫无疑问

```
#1 所有数据都是共享的，这其中，代码作为一种数据也是被所有线程共享的（test.py的所有代码以及Cpython解释器的所有代码）
例如：test.py定义一个函数work（代码内容如下图），在进程内所有线程都能访问到work的代码，于是我们可以开启三个线程然后target都指向该代码，能访问到意味着就是可以执行。

#2 所有线程的任务，都需要将任务的代码当做参数传给解释器的代码去执行，即所有的线程要想运行自己的任务，首先需要解决的是能够访问到解释器的代码。
```

　　综上：

　　如果多个线程的target=work，那么执行流程是

　　多个线程先访问到解释器的代码，即拿到执行权限，然后将target的代码交给解释器的代码去执行

　　解释器的代码是所有线程共享的，所以垃圾回收线程也可能访问到解释器的代码而去执行，这就导致了一个问题:对于同一个数据100，可能线程1执行x=100的同时，而垃圾回收执行的是回收100的操作，解决这种问题没有什么高明的方法，就是加锁处理，如下图的GIL，保证python解释器同一时间只能执行一个任务的代码

　　<img src="https://img2018.cnblogs.com/blog/988061/201809/988061-20180926100512742-1573465285.png" alt="" />

　　

### 三 GIL与Lock

**GIL保护的是解释器级的数据，保护用户自己的数据则需要自己加锁处理，如下图**

<img src="https://img2018.cnblogs.com/blog/988061/201809/988061-20180926100556096-1593917726.png" alt="" />

 

 

### 四 GIL与多线程

　　有了GIL的存在，同一时刻同一进程中只有一个线程被执行

　　听到这里，有的同学立马质问：进程可以利用多核，但是开销大，而python的多线程开销小，但却无法利用多核优势，也就是说python没用了，php才是最牛逼的语言？

　　别着急啊，老娘还没讲完呢。

　　要解决这个问题，我们需要在几个点上达成一致：

```
#1. cpu到底是用来做计算的，还是用来做I/O的？

#2. 多cpu，意味着可以有多个核并行完成计算，所以多核提升的是计算性能

#3. 每个cpu一旦遇到I/O阻塞，仍然需要等待，所以多核对I/O操作没什么用处 
```

　　一个工人相当于cpu，此时计算相当于工人在干活，I/O阻塞相当于为工人干活提供所需原材料的过程，工人干活的过程中如果没有原材料了，则工人干活的过程需要停止，直到等待原材料的到来。

　　如果你的工厂干的大多数任务都要有准备原材料的过程（I/O密集型），那么你有再多的工人，意义也不大，还不如一个人，在等材料的过程中让工人去干别的活，

　　反过来讲，如果你的工厂原材料都齐全，那当然是工人越多，效率越高

　　结论：

　　　　对计算来说，cpu越多越好，但是对于I/O来说，再多的cpu也没用

　　　　当然对运行一个程序来说，随着cpu的增多执行效率肯定会有所提高（不管提高幅度多大，总会有所提高），这是因为一个程序基本上不会是纯计算或者纯I/O，所以我们只能相对的去看一个程序到底是计算密集型还是I/O密集型，从而进一步分析python的多线程到底有无用武之地

```
#分析：
我们有四个任务需要处理，处理方式肯定是要玩出并发的效果，解决方案可以是：
方案一：开启四个进程
方案二：一个进程下，开启四个线程

#单核情况下，分析结果: 
　　如果四个任务是计算密集型，没有多核来并行计算，方案一徒增了创建进程的开销，方案二胜
　　如果四个任务是I/O密集型，方案一创建进程的开销大，且进程的切换速度远不如线程，方案二胜

#多核情况下，分析结果：
　　如果四个任务是计算密集型，多核意味着并行计算，在python中一个进程中同一时刻只有一个线程执行用不上多核，方案一胜
　　如果四个任务是I/O密集型，再多的核也解决不了I/O问题，方案二胜

 
#结论：现在的计算机基本上都是多核，python对于计算密集型的任务开多线程的效率并不能带来多大性能上的提升，甚至不如串行(没有大量切换)，但是，对于IO密集型的任务效率还是有显著提升的。
```

 

### 五 多线程性能测试

 

```
from multiprocessing import Process
from threading import Thread
import os,time
def work():
    res=0
    for i in range(100000000):
        res*=i


if __name__ == '__main__':
    l=[]
    print(os.cpu_count()) #本机为4核
    start=time.time()
    for i in range(4):
        p=Process(target=work) #耗时5s多
        p=Thread(target=work) #耗时18s多
        l.append(p)
        p.start()
    for p in l:
        p.join()
    stop=time.time()
    print('run time is %s' %(stop-start))
```

```
from multiprocessing import Process
from threading import Thread
import threading
import os,time
def work():
    time.sleep(2)
    print('===>')

if __name__ == '__main__':
    l=[]
    print(os.cpu_count()) #本机为4核
    start=time.time()
    for i in range(400):
        # p=Process(target=work) #耗时12s多,大部分时间耗费在创建进程上
        p=Thread(target=work) #耗时2s多
        l.append(p)
        p.start()
    for p in l:
        p.join()
    stop=time.time()
    print('run time is %s' %(stop-start))
```

 

　　应用：

　　　　多线程用于IO密集型，如socket，爬虫，web　　　　多进程用于计算密集型，如金融分析

 
