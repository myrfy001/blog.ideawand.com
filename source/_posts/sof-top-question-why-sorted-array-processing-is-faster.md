---
title: StackOverflow高赞问题解析:为什么这段代码在有序数组上执行比乱序数组快？（附Python版实验）
date: 2019-11-10 18:23:24
tags: Python, 分支预测
---




# 0x00 问题描述
计算数组中大于某个阈值的元素的和，为什么当数组是有序的情况下，代码执行速度是数组无序时的六倍左右？

<!--more-->
> SOF原始问题链接:https://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-processing-an-unsorted-array

问题中所使用的代码如下所示：


```C++
#include <algorithm>
#include <ctime>
#include <iostream>

int main()
{
    // 产生试验用的数组
    const unsigned arraySize = 32768;
    int data[arraySize];

    for (unsigned c = 0; c < arraySize; ++c)
        data[c] = std::rand() % 256;

    // !!! 当加入下面一行代码后，主循环的耗时会减小，而注释掉下面一行代码后，主循环耗时增加
    std::sort(data, data + arraySize);

    clock_t start = clock();
    long long sum = 0;

    for (unsigned i = 0; i < 100000; ++i)
    {
        // 主循环
        for (unsigned c = 0; c < arraySize; ++c)
        {
            if (data[c] >= 128)
                sum += data[c];
        }
    }

    double elapsedTime = static_cast<double>(clock() - start) / CLOCKS_PER_SEC;

    std::cout << elapsedTime << std::endl;
    std::cout << "sum = " << sum << std::endl;
}
```


# 0x01 高赞回答

因为现代CPU在执行指令过程时运用了分支预测技术，对于有序数组，CPU可以通过统计规律来提高分支预测的成功概率，从而避免频繁清洗指令流水线，从而提高执行速度。

#### 如何通俗理解分支预测？
假设你是一个生活在没有电子通信技术时代的铁路扳道工，那么你有两种工作方式：
* 方案一：每过来一辆火车，你都要让司机停下来，问清楚司机要去哪个方向，然后调整道岔
* 方案二：你根据你的经验，提前把道岔调整到一个方向，让火车直接开过去。
  * 如果你猜对了，那么火车就可以不停车快速通过，节省大量时间
  * 如果你猜错了，那么火车司机会在通过道岔后发现错误，然后停车，后退，这时你就知道发生了错误，就可以重新调整道岔，让火车去正确的方向。
  
 如果你能够在绝大多数情况下预测正确，那么显然第二种方案会大幅提高火车通过道岔的效率；反之，如果你预测的不是那么准，那么火车通过的效率就可能和第一种方案差不多（每次都要停下来）


#### CPU中的分支预测是什么东西

对于代码中的if语句、switch语句等，经过编译器转换为汇编语言后，对应的都是`分支指令（branch instruction）`，现代CPU通常都会尝试从最近执行的一些列指令中学习某些“经验数据”，从而提高自己预测分支的正确率。

#### 如何解释上述问题中的现象

对于排序的数组的前一段数据，由于判断条件会一直不成立，因此CPU的分支预测机制会学习到这个特征，并且每次假设条件不成立，而当进入到数组后半段的处理时，CPU也能很快发现每次循环判断条件都是成立的，从而调整自己的预测行为。也就是说，对于排序数组而言，绝大部分情况下，分支预测都可以生效，每次处理流水线都不需要被重新清洗（也就是火车不用停车，后退，再前前进），因而执行速度会很快。

对于没有排序的数组，由于每次判断条件会随机的成立或者不成立，从而导致CPU无法学习到一个预测分支的模式，从而导致CPU进行分支预测时经常出现失误，也就会导致CPU内部的流水线会不断被刷洗再重新装填，导致无法发挥CPU的最大性能。


# 0x02 Python版本实验
我们编写如下的Python程序，看看能不能在Python语言中复现上述问题。

```Python

import time
import random

def test():
    data = [random.randint(0,256) for _ in range(32768)]
    
    # 尝试注释或打开下面的语句，观察效果
    data.sort()

    t0 = time.time()
    sum = 0
    for _ in range(100000):
        for v in data:
            if v >= 128:
                sum += v
    t1 = time.time()

    print(t1-t0)

test()

```

如果您尝试运行上述代码，就会发现好像在两种情况下，耗时没有明显差异。之所以看不到明显差异的原因，是因为Python是一门解释型语言，而非编译型语言。Python解释器的核心是一个巨大的switch语句(代码参见ceval.c),而我们一个简单的Python语句实际上会被转换为多个Python字节码，通过多次switch语句才能执行，也就是说，为了执行我们的上述Python代码，CPU实际上要处理多次分支场景，已经把CPU搞蒙了，分支预测的功效无法发挥出来。从这里大家也可以更深一层来理解为什么大家说Python解释器“慢”了。


那么，真的无法用Python语言来复现上面这个问题吗？答案是否定的，只不过我们需要换一个Python解释器实现，在这里，我们用PyPy来执行上面的代码，就可以看到明显的差异:排序后的执行时间大约是乱序执行时间的一半。

想进一步了解PyPy是什么，想看上述代码用pypy加速的具体实现？关注公众号，回复pypy获取文章。

用Python正则表达式处理带有注音符号的Unicode文本会有坑吗？点击这里


关于作者：在校玩嵌入式，机缘巧合进入互联网IT行业工作，对各种底层原理有浓厚兴趣，Python工程师。长按下方二维码关注我的公众号，一起探索那些增删改查之外的底层技术。