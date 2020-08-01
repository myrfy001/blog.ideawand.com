---
title: 实战:用Python实现tail命令
date: 2020-03-22 11:22:05
tags: python 文件系统 tail命令
---

# 背景介绍
`tail`命令作为linux下最常用的命令之一，大家肯定不陌生，`tail -f`用法更是排查问题查看日志的利器。很多日志收集工具，比如logstash、filebeat、flume等，也都能实现`tail -f`的类似功能，从文件末尾不断读取新的内容，并发送到指定的收集服务集群。那么，要实现这样一个工具，我们要用到哪些与文件操作有关的知识呢？

笔者曾经遇到这样的场景，公司采用zipkin来追踪各个系统之间的调用关系，每个子系统产生自己的日志并落盘，由每台服务器上部署的flume来收集日志并收集上报。

那么就有了一个问题，线上环境有公司的整套日志系统，而开发环境比较简陋，并没有部署收集工具。考虑到flume和logstash都是java系，有点重，而filebeat有个坑就是上报数据它一定会自作主张给外面套上一层结构，导致收集集群无法识别，于是笔者就想，能不能用python脚本，争取在50行代码之内，实现一个日志收集上报工具呢？

我们要解决如下几个问题:
* 如何快速定位到文件最后的n行？
* 日志文件可能发生切割，如何保证日志切割后还能正确读取最新的文件？
* 如何高效的尽快检测到文件写入？


<!--more-->

# 如何快速定位到日志最后N行？

## 方法1
在不考虑性能的前提下，如何暴力的定位到日志最后一行呢？ Python中文件对象的`readline()`方法在到达文件结尾后会返回空字符串，利用这个特性，我们可以在打开文件后，通过一个暴力的while循环来到达文件的最后一行，就像这样：
```python
with open("/tmp/a.log", "r") as fi:
    while(fi.readline()):
        pass
    # 到达此处时，文件指针已经指向文件末尾了
```

## 方法2
如果日志文件很大，上述的方式肯定不靠谱了。那么如何跳转到文件的最后呢？通过使用`fi.seek(0, io.SEEK_END)`可以实现将当前文件描述符的指针移到文件结尾，如下所示：
```python
import io
with open("/tmp/a.log", "r") as fi:
    fi.seek(0, io.SEEK_END)
```

但这样有一个问题，由于日志写入程序和我们的读取程序位于两个进程中，而两个进程之间并没有加锁等同步机制，因此我们所跳转到的位置可能是一行日志写了一半的中间部分，那么如何才能对齐到一行的开始呢？有两种思路，一种是继续等待文件追加，直到检测到下一个\n为止，另一种方式则比较复杂，我们可以向前回退文件指针，去搜寻前面的换行符。那么我们来挑战一下这个高难度的实现方式。

算法的难点在于，我们要考虑程序的性能，例如我们在向前查找的时候对内存的要求等都希望有一定的限制，避免过大的日志文件消耗内存过大。直接上代码~
```python
# 传入指定的文件对象和希望读取的行数，将文件指针移动到倒数第n行的开头位置
# 代码作者 http://blog.ideawand.com 微信公众号：极客幼稚园 转载请注明
def seek_to_last_n_line(fi, n, buf_size=4096):

    now_line_cnt = 0
    output_start_pos = 0

    # 如果是空文件，那么直接返回空列表即可
    read_pos = fi.seek(0, io.SEEK_END)
    if read_pos == 0:
        return []

    # 检查最后一行是否以换行结尾。这里涉及到多一行还是少一行的边界情况处理
    fi.seek(read_pos-1)
    x = fi.read(1)
    if x == b"\n":
        now_line_cnt = -1

    # 算法主体部分，每次读取一个指定缓冲区大小的数据块，然后寻找换行符的个数
    run = True
    while run:
        read_pos = max(read_pos-buf_size, 0)
        fi.seek(read_pos)
        data = fi.read(buf_size)
        end_pos = len(data)

        # 在读取的缓冲区中从右向左查找换行符
        while 1:
            # rfind在底层是解释器用C语言实现的，效率非常高
            new_line_pos = data.rfind(b"\n", 0, end_pos)
            if new_line_pos == -1:
                if read_pos == 0:
                    run = False
                break

            end_pos = new_line_pos
            now_line_cnt += 1
            output_start_pos = read_pos + new_line_pos + 1
            if now_line_cnt == n:
                run = False
                break

    # 如果整个文件遍历到开头都没有满足行数要求，那就输出整个文件
    if now_line_cnt < n:
        output_start_pos = 0

    # 代码来自 http://blog.ideawand.com 微信公众号：极客幼稚园 转载请注明
    fi.seek(output_start_pos)
    
```
> 注意上面的算法中传入的文件对象一定要用`"rb"`，也就是二进制的方式打开文件，这是因为：
> * 一方面在算法中我们用到了`seek`方法，有可能落到多字节字符（例如汉字）编码的中间，如果使用文本方式打开会导致decode失败。
> * 另一方面，文本模式打开会导致Python解释器为我们做byte到unicode的decode操作，增加了开销。


# 如何检测日志发生切割？
如何检测一个文件是否发生了日志切割呢？通常的日志切割工具会首先把老的日志文件重命名，然后创建一个同名的新文件。其实大家会发现，无论日志收集程序，还是日志写入程序，都需要实现区分文件是否发生切割的功能。这个功能的原理是linux文件系统下，同一个分区中每个文件都会有一个inode编号，而不同的磁盘分区会有不同的设备编号（dev_id）,所以，我们只要通过比较当前打开的文件和通过文件路径索引到的文件是否具有相同的上述id即可实现这个功能：

* 对于当前打开的文件对象，通过`os`模块中的`fstat()`函数可以获得文件信息
* 对于已知路径但并没有打开的文件，通过`os`模块中的`stat()`函数可以获取文件信息

简要写一下代码的实现框架：
```python
file_to_monit = "/tmp/aa.log"
fd = open(file_to_monit)
while 1:
    if os.fstat(fd.fileno()).st_ino != os.stat(file_to_monit).st_ino:
        fd.close()
        fd = open(file_to_monit)
```

不过，很明显可以看出，上面的代码并不能在实际中使用，因为这个while循环过于消耗资源了。我们应该尽量减少对文件变更的检测次数，仅在需要的时候再进行检测。那么应该如何定义这个“需要的时候”呢？大家可以先想一想。我们会在下一节中介绍一种解决方案。

# 如何快速检测日志追加？
上一节我们讨论的是文件发生了切割的情况，另外你，在文件没有发生切割时，如何高效读取新写入的内容也是一个值得考虑的问题。例如，最暴力的方式可能是下面这样：
```python
while 1:
    new_line = fi.readline()
    # 如果读到文件结尾，readline()会返回空字符串
    if not new_line:
        continue
    # 在这里做和日志处理相关的内容
```

上述代码的确可以实现检测文件是否有追加写入内容，但会直接将CPU使用率打到100%，可以说并不是一个可以实际使用的方式。那么，我们来做一点优化，如果读到了文件结尾，那么就睡眠一段时间，然后再检测下一次：
```python
import time
while 1:
    new_line = fi.readline()
    if not new_line:
        # 睡眠1秒钟，避免CPU占满
        time.sleep(1)
        continue
```

上面的方式，可以说是一种最简单，并且非常实用的解决手段，毕竟大多数场合下，我们日志离线收集有一秒钟的延时是完全可以接受的。知名的日志收集工具filebeat采用的就是这种轮工具。

如果到此已经满足了你的好奇心，那么就可以不继续阅读了。不过，作为有追求的开发者，你一定想要继续优化我们的实现方式，做到尽可能实时发现日志的更新，又不占用额外的CPU资源。那怎么才能实现呢？

假设你对Linux系统，特别是Linux网络编程很熟悉的话，那你一定知道Linux下为了在Socket上实现高性能的读写等待，引入了select、poll、epoll等机制，而Linux下，Socket和普通文件一样，都具有文件描述符，那么是不是可以用select之类的机制来监测文件的变化呢？我们来试一下：

```python
# coding:utf-8
import select

file_to_monit = "/tmp/aa.log"
fd = open(file_to_monit)

while 1:
    r,w,e = select.select([fd], [], [], 1)
    if not r:
        print("no more data")
        continue
    line = fd.readline()
    print(line)
```
各位同学如果亲自试一下，就会发现上面的程序并没有得到我们期望的结果。实际上，Linux的select等IO复用工具是不能作用在普通文件上的，具体的原因，大家可以去搜索一下，在这里不展开讨论了。
为了实现我们想要的功能，正确的做法是使用Linux内核提供的inotify机制。为了简单起见，我们可以直接使用已经包装好的inotify库来完成我们的任务，inotify机制的具体原理，大家也可以去网上搜一下。
我们先执行 `pip install inotify`来安装访问inotify接口的库

```python
# coding:utf-8
import select
import inotify.adapters


file_to_monit = "/tmp/aa.log"
fd = open(file_to_monit)

i = inotify.adapters.Inotify()

i.add_watch(
    file_to_monit, 
    mask=inotify.constants.IN_MODIFY # 仅监听更新事件
)

while 1:
    for event in i.event_gen():
        line = fd.readline()
        if not line:
            # 类似select,inotify也有一个超时时间，如果超时时间内没有发生写入，则我们仍然会读到一个空结果
            continue
        print(line)
```

最后，要提醒大家一下，inotify是Linux内核提供的系统调用，存在跨平台的问题，如果要在Mac OsX 或者Windows下使用的话，可以考虑安装python的watchdog包，这个包会在不同的平台下为你自动选择平台支持的方式来完成上述工作。