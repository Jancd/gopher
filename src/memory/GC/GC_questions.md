## GC信息的信息收集
设置环境变量**GODEBUG=gctrace=1**。

使用方法，如果程序为myserver。正常的启动方法为./hello，如果需要收集GC信息启动方式**如下`GODEBUG=gctrace=1  ./hello` 或者 `GODEBUG=gctrace=1 go run hello.go`。**

### GC信息分析
```
gc5(6): 11+12+357+77 us, 0 -> 1 MB, 4294 (5261-967) objects, 67/2/0 sweeps, 6(115) handoff, 6(9) steal, 170/56/5 yields
```

`gc5(6)`：表示第5次GC，共有6个线程参与GC。

`11+12+357+77 us`：表示停止各个goroutine花费时间是11us，释放标记对象所有时间为12us，扫描标记可回收对象花费时间为257us，完成各个线程结束为17us。GC总时间为457us。

`0 -> 1 MB`：表示上次GC后堆占用的空间为0MB，本次GC前堆占用的空间为1MB。

`4294 (5261-967) objects`：表示剩余未释放对象个数为4294个，GC之前拥有对象为5261个，本次GC释放对象967个。

基本规律是当前对象越多，扫描时间越长，需要释放的对象越多，释放过程越长。

## GC 优化小技巧

减少对象分配 所谓减少对象的分配，实际上是尽量做到，对象的重用。 比如像如下的两个函数定义：


```go
func(r*Reader)Read()([]byte,error) //不建议！！
func(r*Reader)Read(buf[]byte)(int,error)
```

第一个函数没有形参，每次调用的时候返回一个 `[]byte` ，第二个函数在每次调用的时候，形参是一个 `buf []byte` 类型的对象，之后返回读入的`byte`的数目。

第一个函数在每次调用的时候都会分配一段空间，这会给 gc 造成额外的压力。**第二个函数在每次调用的时候，会重用形参声明**。

#### string 与 []byte 转化

在 stirng 与 []byte 之间进行转换，会给 gc 造成压力 通过 gdb ，可以先对比下两者的数据结构：


```go
type = struct []uint8 {    uint8 *array;    int len;    int cap;}
type = struct string {    uint8 *str;    int len;}
```

两者发生转换的时候，底层数据结结构会进行复制，因此导致 gc 效率会变低。解决策略上，一种方式是一直使用 []byte ，特别是在数据传输方面，[]byte 中也包含着许多 string 会常用到的有效的操作。另一种是使用更为底层的操作直接进行转化，避免复制行为的发生，使用 `unsafe.Pointer` 直接进行转化。

**尽少使用** `+` 连接 string ！！！ 由于采用 `+` 来进行 string 的连接会生成新的对象，降低 gc 的效率，**好的方式是通过 append 函数来进行**。

#### 关于append

参考如下代码：


```go
b := make([]int, 1024) 
b = append(b, 99) 
fmt.Println("len:", len(b), "cap:", cap(b))
```

在使用了append操作之后，数组的空间由 1024 增长到了 1312 ，所以如果能提前知道数组的长度的话，**最好在最初分配空间的时候就做好空间规划操作**，会增加一些代码管理的成本，同时也会**降低 gc 的压力，提升代码的效率**。

对上面的代码可以这样改进：


```go
b := make([]int, 0, 1024)
```


## 触发GC机制

1. 在申请内存的时候，检查当前当前已分配的内存是否大于上次GC后的内存的2倍，若是则触发（主GC线程为当前M）

2. 监控线程发现上次GC的时间已经超过两分钟了，触发；将一个G任务放到全局G队列中去。（主GC线程为执行这个G任务的M）

Go runtime底层也采用内存池，但每个span大小为4k，同时维护一个cache。

cache有一个0到n的list数组，list数组的每个单元挂载的是一个链表，链表的每个节点就是一块可用的内存，同一链表中的所有节点内存块都是大小相等的；但是不同链表的内存大小是不等的，也就是说list数组的一个单元存储的是一类固定大小的内存块，不同单元里存储的内存块大小是不等的。

这就说明cache缓存的是不同类大小的内存对象，当然想申请的内存大小最接近于哪类缓存内存块时，就分配哪类内存块。当cache不够再向spanalloc中分配。

## 为什么小对象多了会造成gc压力

- 每个对象创建的时候就意味着没存分配，小对象多了意味着内存分配次数也多了。
- 小对象在堆上频繁地申请释放，会造成内存碎片（有的叫空洞），**导致分配大的对象时无法申请到连续的内存空间**
- 如果减少了内存的分配是不是可以减少GC的次数

**Go runtime底层也采用内存池，但每个span大小为4k，同时维护一个cache。cache缓存的是不同类大小的内存对象，当然想申请的内存大小最接近于哪类缓存内存块时，就分配哪类内存块。**

**go 1.8甚至量坏情况下GC为100us**
