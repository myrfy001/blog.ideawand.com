---
title: Golang GC调优实战系列之一
date: 2020-08-01 22:32:29
tags: golang go GC 实战
---

本文记录了笔者在工作中遇到一个真实场景，在堆内存超过16G,堆对象过亿情况下，以真实案例带领读者感受不同参数设置情况下Go的GC性能变化。

<!--more-->

# 背景
笔者所维护的服务在启动过程中需要加载一个巨大的词表到内存中，词条数量达到千万量级，加之其他辅助数据结构，堆对象数量达到过亿级别。

笔者开发机为16G内存，起初词表体积不大，在开发机可以顺利启动调试，随着词表不断加大，程序启动时所占用的内存已超出物理内存大小，导致程序无法启动（笔者没有开启交换分区）。

另一个有用的背景信息是，可以观察到程序在启动时所占用的内存是最大的，程序启动后，会逐渐将申请的内存归还给操作系统，并最终稳定在某个数值上，而这个最终稳定的数值大约是启动时峰值内存的60%左右。

根据上述现象，推断这是由于启动时垃圾回收不及时导致的，通过调节Go的垃圾回收控制参数，笔者成功控制了服务启动时所需要使用的内存大小。

# 核心代码

使用下述程序来抽象公司的业务逻辑，消费者从队列中读取序列化的数据，将其反序列化到一个临时的中间数据结构中，将变形后的数据写入map中保存，抛弃中间结果。可以想到，造成内存占用量大的问题在于这些临时的中间对象被抛弃后，没有及时被GC回收。

```golang
package main

import (
	"encoding/json"
	"fmt"
	"runtime"
	"sync"
)

// 用于解码数据的临时结构
type QMessage struct {
	ID   uint64       `json:"id,omitempty"`
	Body QMessageBody `json:"body,omitempty"`
}
type QMessageBody struct {
	Field1 string `json:"field_1,omitempty"`
	Field2 int    `json:"field_2,omitempty"`
}

// 常驻于内存的数据结构
type Message struct {
	id     uint64
	field1 string
	field2 int
}

// 内存数据缓存，里面存放了千万级的词条信息
var buffer = make(map[int]Message)


func main() {
	wg := &sync.WaitGroup{}
	q := make(chan string)
	wg.Add(2)
	go producer(q, wg)
	go consumer(q, wg)
	wg.Wait()
	PrintMemUsage()

}

// 模拟生产者，产生两千万词条数据
func producer(q chan string, wg *sync.WaitGroup) {
	for i := 0; i < 20000000; i++ {
		q <- `{"id":123456, "body":{"field1": "123", "field2": 456}}`
	}
	close(q)
	wg.Done()
}

// 模拟消费者，消费并反序列化词条数据，使用中间临时数据结构进行数据变形，并将最后的结果存储
func consumer(q chan string, wg *sync.WaitGroup) {
	idx := 0
	for data := range q {
		idx++
		qtmp := QMessage{}
		json.Unmarshal([]byte(data), &qtmp)

		tmp := Message{
			id:     qtmp.ID,
			field1: qtmp.Body.Field1,
			field2: qtmp.Body.Field2,
		}
		buffer[idx] = tmp
	}
	wg.Done()
}

// 以下是打印内存监控数据的工具函数，与业务逻辑无关
func PrintMemUsage() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)

	// For info on each, see: https://golang.org/pkg/runtime/#MemStats
	fmt.Printf("Alloc = %v MiB", bToMb(m.Alloc))
	fmt.Printf("\tTotalAlloc = %v MiB", bToMb(m.TotalAlloc))
	fmt.Printf("\tSys = %v MiB", bToMb(m.Sys))
	fmt.Printf("\tNumGC = %v", m.NumGC)
	fmt.Printf("\tAllocObjCnt = %v", m.Mallocs)
	fmt.Printf("\tSTW = %v\n", m.PauseTotalNs)

}

func bToMb(b uint64) uint64 {
	return b / 1024 / 1024
}

```
# 调优&测试

Go为开发者只暴露了一个调节GC的参数，这个参数可以通过在启动Go之前指定环境变量`GOGC`来控制，他控制的是触发GC的阈值，是一个百分比。当Go新创建的对象所占用的内存大小，除以上次GC结束后保留下来的对象占用内存大小，所得到的比值大于`GOGC`设置的阈值时，就会触发一次GC。如果没有指定这个变量，则默认值是100。也就是说，默认情况下，当目前占用内存是上次GC结束后占用内存的一倍时，才会触发一次GC。

我们的优化思路到这里很明显了：
* 降低触发GC的阈值，避免等待GC的对象占用过多的内存
* 降低阈值势必增加GC的次数，那么需要对GC次数增加带来的性能损失做一下评估

为了测试不同GC阈值情况下，程序的性能表现，我们使用下面的脚本来批量测试不同情况下的程序性能：
```shell
go build -o a.run main.go 
data=( -1 12 25 50 100 200)
for i in ${data[@]} ; do
echo "==== start", GOGC=$i "===="
GOGC=$i time -p ./a.run
echo
done
```

# 调优数据分析

测评数据如下表所示，第一行`GOGC=-1`表示禁止完全GC，第二行`GOGC=12`表示新分配对象占用内存超过上次GC后程序占用内存的12%时会触发一次新的GC。

|GOGC| 物理内存占用 | 程序执行时间 | 用户态CPU时间 | 内核时间 | GC次数 | GC STW时间  | 
|----|------------|------------|-------------|---------|-------|-------------|
|-1  | 9165 MB    | 38.52 s    | 37.01 s     | 5.25 s  | 0     | 0           |
|12  | 1507 MB    | 39.04 s    | 66.31 s     | 3.30 s  | 223   | 14168116 ns |
|25  | 1494 MB    | 37.58 s    | 51.11 s     | 3.33 s  | 121   | 7355709  ns |
|50  | 1623 MB    | 36.76 s    | 44.30 s     | 3.15 s  | 65    | 3817951  ns |
|100 | 4188 MB    | 37.50 s    | 40.94 s     | 4.27 s  | 31    | 1653426  ns |
|200 | 4062 MB    | 37.73 s    | 40.37 s     | 4.46 s  | 20    | 1062916  ns |

观察表格，我们可以有以下几个结论（不一定严谨，但从数据上看大方向是对的）
* 对比前两行，即完全禁用GC和开启GC的情况，可以看到，我们的代码大约产生了7.5G的中间对象
* 对比`GOGC=12`和`GOGC=100`两行,可以看到，经过调优的内存消耗降到了默认设置的大约三分之一
* 观察`GC STW时间`和`GC次数`两列，可以发现，Go使用三色标记算法的GC Stop The World停顿时间还是非常短的。
  * 可以在上述程序中增加打印`m.PauseNs`来观察每次gc停顿的耗时，基本每次停顿时间不会超过100微秒，也就是0.1毫秒
* 观察`程序执行时间`和`用户态CPU时间`,可以发现，增加GC次数后，程序执行的耗时没有明显增加，但CPU时间增长很多。
  * 说明GC的扫描过程充分利用了多核的优势，GC的主体过程并不会明显block用户逻辑的执行。
  * 从这一点可以加深对Go GC不同阶段的理解。Go的GC只需要在首尾两个很短的环节中引入StopTheWorld的停顿，而其他大部分GC的时间都是与用户逻辑并行的
  * 如果你的机器有充足的空闲CPU时间给GC任务使用，则对用户逻辑的影响很小
* 关闭GC后，按照我们的直观理解，减少了GC的次数，应该执行耗时更短才对，不过通过观察上表可以看出，关闭GC后，内核时间消耗明显增加，同时程序运行时间并没有缩短，甚至比打开GC还要长。
  * 我们可以粗略认为这是由于Go的运行时需要找操作系统分配更多内存导致的。
    * 上述数据跑在16G内存，并且打开交换分区的机器上，机器上同时还运行着其他的应用，关闭GC后占用9G内存，操作系统内核极有可能动用了交换区，进一步拖慢速度
  * 另一方面，由于每个对象都要占用新的内存，导致CPU的Cache命中率降低，也会拖慢运行速度。

# 结论
通过上述数据分析，我们最终选择了设置`GOGC=50`，该取值可以在GC消耗与内存消耗之间取得较好的平衡。