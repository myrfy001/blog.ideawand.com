---
title: pypy简要介绍(附Python代码加速对比实验)
date: 2019-11-11 23:36:09
tags: python pypy
---

一个用Python实现的Python解释器？真的这么简单吗？

<!--more-->

如果大家在网上搜索什么是Pypy，截止2019年11月16日，你会发现百度百科收录的简介是:
>PyPy是用Python实现的Python解释器

而维基百科给出的简介是:

> PyPy is an alternative implementation of the Python programming language to CPython (which is the standard implementation). PyPy often runs faster than CPython because PyPy is a just-in-time compiler while CPython is an interpreter. Most Python code runs well on PyPy except for code that depends on CPython extensions, which either doesn't work or incurs some overhead when run in PyPy. Internally, PyPy uses a technique known as meta-tracing, which transforms an interpreter into a tracing just-in-time compiler. Since interpreters are usually easier to write than compilers, but run slower, this technique can make it easier to produce efficient implementations of programming languages. PyPy's meta-tracing toolchain is called RPython.

在此，不得不说，做技术的同学，还是要趁早放弃万事百度的习惯，如果有Google最好，没有Google的也要用微软的Bing...

好了，现在切入正题，不要被百度百科给出的答案误解了。在维基百科给出的解释中，我们要重点关注一个短语：`just-in-time compiler`，也就是`即时编译器`。此外，维基的简介中还有一个不太显眼的东西`RPython`。关于PyPy和RPython的渊源，我们暂且不做论述（后续或许会专门写一篇文章），但要澄清的一点是，历史上广义的PyPy实际上是一个项目的名称，而我们通常说的PyPy，是指狭义的`PyPy解释器`，现在官方文档中已经将PyPy明确定义为PyPy解释器，本文将简要介绍的也只是PyPy解释器，并提供一个用PyPy解释器来加速Python运行的例子。（下文中的PyPy如无特殊说明，均指代PyPy解释器）

PyPy解释器与最常见的CPython解释器相比，最明显的特征就是PyPy解释器内置了JIT(just-in-time compiler), 内置这个东西有什么用处呢？这要从CPython解释器和Python语言的特性说起。

众所周知，Python是一门动态类型语言，也就是说，任何一个变量，都在运行时才能获取这个变量的类型。那么这就意味着，即使进行一个1+1这样的简单操作，Python的解释器也要去做很多额外的工作，比如用来确认这两个变量是什么类型的，能不能支持`+`运算。这样一来，原本一个CPU硬件指令就可以搞定的事情（对应汇编add），在Python上不知道要用多少个硬件指令才可以完成，所以说Python性能低，速度慢。那么有没有方法，能让Python的解释器在执行的时候，不要每次都去判断类型呢？甚至，对于1+1这样的操作，能不能变成机器指令去完成呢？

上述问题的答案是，可以，但是实现起来不友好。为什么不友好呢？因为动态语言和静态类型本身就是矛盾的，为了实现将a+b转换为机器指令直接去执行，就必须提前知道参与运算的两个变量a和b的类型，如果每次写代码之前都要指定a和b的类型，那Python就不是那个大家用着都说好，编写起来简单流畅的Python了。

写到现在，大家可以感觉到，这个问题的核心是，大家既想要Python书写简单，不用声明类型的特点，又想要静态语言能直接把基础操作翻译成机器指令的能力。怎么解决呢？让大家手动去写类型，这不够Pythonic! 能不能让机器自动帮我们推测出类型呢？嗯！PyPy的一个主要功能就是用来干这件事的。

那么，PyPy是通过什么方式来推断出变量类型的呢？答案是一种叫做`tracing just-in-time compiler`的技术，通俗来说，就是在PyPy运行初期，解释器会统计代码中每个变量的取值情况，根据统计概率，来推断每一个变量的类型。举个例子，对于下面的代码，循环会重复执行100次，我们并没有标记出变量i和sum是一个整形数，但在前N次执行时，PyPy解释器发现每次循环后i、sum的类型都是整数，而整数加减是可以通过简单的机器指令来实现的，于是PyPy解释器为这段程序现场生成机器代码完成这段运算，并在后面的循环中，直接使用机器指令代替解释器的`取指令->类型检查->执行加法操作`这一些列复杂耗时的操作。

```python
sum = 0
for i in range(100):
    if i % 2 == 0:
        sum = sum + i
    else:
        sum = sum + i * 2
```

通过上面的示例，我们可以认识到，要想让PyPy解释器在执行过程中充分发挥JIT的效果，就要尽可能少使用Python的动态特性，尽量保证一个变量名只对应一种数据类型。例如下面这段Python代码，即使使用了PyPy，也无法获得很好的效率提升，因为data的类型每次都在变化，PyPy无法推断其到底是字符串还是整数，因此只能使用解释执行的方式，无法发挥JIT的功效：

``` Python
data = 0
for i in range(100):
    if i % 2 == 0:
        data = i
    else:
        data = str(i)
```

细心的读者可能会问了，PyPy有没有可能做出错误的判断呢？比如下面这段代码，会不会导致最后data的值有问题呢？

``` Python
data = 0
for i in range(100):
    if i < 99:
        data = i
    else:
        data = str(i)
print(type(data))
```

上述这段代码，前99次data都是数值，而最后一次，data变成了字符串，这会不会导致PyPy生成的机器指令在最后一次循环中发生错误呢？其实，PyPy本身在生成机器码的过程中，并不是简单的把Python代码翻译成机器码，而是会额外插入一些校验逻辑，当发现JIT得到的机器指令在某些场景下不正确时在fallback到解释执行字节码的方法上来，而这些校验逻辑由于也是以机器码的形式存在的，效率要远高于Python解释执行时做类型判断。

上面说了这么多，那PyPy的效果到底怎么样呢？如何才能安装PyPy并进行试验呢？我们现在来带领大家实际体验一下。

本次试验基于本公众号【极客幼稚园】之前的一篇文章《StackOverflow高赞问题解析:为什么这段代码在有序数组上执行比乱序数组快？（附Python版实验）》中提到的例子来进行说明。

首先是安装PyPy，这个很简单，到官网(https://pypy.org/download.html)下载自己电脑对应版本，然后解压缩即可使用，必要的话，可以将解压后bin/pypy3这个可执行文件添加到PATH环境变量中。 使用PyPy和CPython唯一的区别是启动方式不同。

将下面的代码保存到/tmp/1.py

```Python
import time
import random

def test():
    data = [random.randint(0,256) for _ in range(32768)]
    
    # 尝试注释或打开下面的语句，观察效果
    data.sort()

    t0 = time.time()
    sum = 0
    for _ in range(1000):
        for v in data:
            if v >= 128:
                sum += v
    t1 = time.time()

    print(t1-t0)

test()

```


下面来进行对比试验， 为了屏蔽个体差异，我们使用PyPy和CPython解释器各执行5遍，同时为了复现在《StackOverflow高赞问题解析:为什么这段代码在有序数组上执行比乱序数组快？（附Python版实验）》这篇文章文末留下的小悬念，我们对数组是否预先排序做两组实验：

```Shell
# CPython解释器，先对数组进行排序
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.7666270732879639
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.7406818866729736
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.727403163909912
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.7232551574707031
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.7416167259216309

# PyPy解释器，先对数组进行排序
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.13713502883911133
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.12418985366821289
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.12426996231079102
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.13276219367980957
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.1257169246673584

# CPython解释器，没有预先排序
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.876326084136963
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.8977439403533936
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.88326096534729
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.9016320705413818
[~/pypy3.6-v7.2.0-osx64/bin]> python /tmp/1.py
1.9013550281524658


# PyPy解释器，没有预先排序
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.24656891822814941
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.25657105445861816
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.24218487739562988
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.24288010597229004
[~/pypy3.6-v7.2.0-osx64/bin]> ./pypy3 /tmp/1.py
0.2552320957183838
```


对于实验现象，相信读者可以做出自己的解释。