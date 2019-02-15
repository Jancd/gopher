## 关于服务编排：

编排能够很大程度上简化分布式服务调用的复杂度，如同步、异步、异步模拟同步、超时重试、事务补偿等，均有服务编排引擎完成，不再完全依赖程序员老师傅的编码能力。

# Golang面试问题汇总:

1.说说为什么同样实现一个“hello world”，Go编译出来的程序一般会比C/C++要大？
关键字：跨平台 静态链接编译 依赖库 自带runtime

2.说说channel的实现.（核心，拓展问题：通信常用手段，阻塞非阻塞，同步异步的区别，select／poll/epoll等等）

3.goruntine是怎么调度的？与进程，线程的关系。（核心，拓展问题，进程，线程，协程区别，死锁，操作系统等等）

4.如何理解“不要通过共享内存来通信，要通过通信来共享内存”这句话？（关于代码和设计）

关键字：高内聚低耦合 消息机制 channel 结合场景

5.说说排查Go问题的经历，都用到了什么工具，有什么看法？（经验，排查问题思路和能力，个人觉得排查问题重要的是有没有思路方法，不见得什么最有效）

6.Go目前的GC策略是什么？之前是什么？怎么改进的？（拓展问题：关于GC算法，内存分配等等）


## 关于 goroutine 的退出：

那么我们怎么来控制一个goroutine，使它可以结束自己的使命正常结束呢？其实很简单，同样我们使用`select + return`来实现这个功能。

```go
func boring(msg string, quit chan bool) <-chan string { 
    c := make(chan string)
    go func() { 
        for i := 0; ; i++ {
        	select {
        	case c <- fmt.Sprintf("%s: %d", msg, i):
        		time.Sleep(time.Duration(rand.Intn(1e3)) * time.Millisecond)	
        	case <-quit:
        		return
        	}
        }
    }()
    return c
}
func main(){
	quit := make(chan bool)
    c := boring("Joe", quit)
    for i := rand.Intn(10); i >= 0; i-- { fmt.Println(<-c) }
    quit <- true
}
```

## Go语言对网络IO的优化

主要是从两方面下手的：

- a. 将标准库中的网络库全部封装为非阻塞形式，防止其阻塞底层的M并导致内核调度器切换上下文带来的系统开销。
- b. 运行时系统加入epoll机制(针对Linux系统)，
>当某一个Goroutine在进行网络IO操作时，如果网络IO未就绪，就将其该Goroutine封装一下，放入epoll的等待队列中，当前G挂起，与其关联的M可以继续运行其他G。

>当相应的网络IO就绪后，Go运行时系统会将等待网络IO就绪的G从epoll就绪队列中取出（主要在两个地方从epoll中获取已网络IO就绪的G列表，一是sysmon监控线程中，二是自旋的M中），再由调度器将它们像普通的G一样分配给各个M去执行。

Go语言将高性能网络IO的实现方式直接集成到了Go本身的运行时系统中，与Go的并发调度系统协同高效的工作，让开发人员可以简单，高效地进行网络编程。

## 1. 除了 mutex 以外还有那些方式安全读写共享变量？
    
A。封装一些线程读写安全的类型作为共享变量，如`Atomic`。

B，将共享变量的读写放到一个 goroutine 中，其它 goroutine 通过 channel 进行读写操作。

c,最常见的互斥了，共享变量要互斥访问:
```go
    sema <- struct{}{} // acquire token
    balance = balance + amout
    <-sema // release token
```

## 2. 无缓冲 chan 的发送和接收是否同步?

当通道中有数据时，向通道中发送会阻塞，当channel中无数据是，从channel中接收会阻塞。是否同步取决于使用者的操作。

同步的概念是：执行一个函数或者方法后，必须等到返回信息才能进行下一步操作。**从这方面理解的话，无缓冲的channel可以理解为同步的。因为发送必须等待channel中有数据，channel接收必须等待channel中无数据**


## 3. go语言的并发机制以及它所使用的CSP并发模型．

golang的并发建立在 goroutines，channels，select等的基础设施之上

csp并发模型：CSP 描述这样一种并发模型：多个Process 使用一个 Channel 进行通信,  这个 Channel 连结的 Process 通常是匿名的，消息传递通常是同步的。

go 其实只用到了 csp 模型的一小部分，即理论中的 **Process/Channel（对应到语言中的 goroutine/channel）**：这两个并发原语之间没有从属关系， Process 可以订阅任意个 Channel，**Channel 也并不关心是哪个 Process 在利用它进行通信**；Process 围绕 Channel 进行读写，形成一套有序阻塞和可预测的并发模型。


## 4. golang 中常用的并发模式？

- 通过sync包中的WaitGroup实现并发控制
- Context上下文，实现并发控制

>根据 Rob pike 在 Google I/O 2012上的[演讲](http://talks.golang.org/2012/concurrency.slide#55)，golang的并发建立在 goroutines，channels，select等的基础设施之上，总结起来主要是：（这些都是基于csp并发模型进行的）
- 生产者消费者模型（多对一）
- 发布订阅模型（多对多m:n）【在这个模型中，消息生产者成为发布者（publisher），而消息消费者则成为订阅者（subscriber），生产者和消费者是M:N的关系】

## nil slice和empty slice的区别

以下是错误的用法，==会报数组越界的错误==，因为只是声明了slice，却没有给实例化的对象，这一点如果是cpp的vector，便可以直接使用，但是golang 不行。


```
var slice []int
slice[1] = 0
```

此时**slice的值是nil**，这种情况可以用于需要返回slice的函数，当函数出现异常的时候，保证函数依然会有nil的返回值。

empty slice 是指slice不为nil，但是slice没有值，**slice的底层的空间是空的**，此时的定义如下：


```
slice := make([]int,0）//或者
slice := []int{}
```


## 5. JSON 标准库对 nil slice 和 空 slice 的处理是一致的吗？　
**==不一致==！！！**

```go
package main

import (
	"encoding/json"
	"fmt"
)

type A struct {
	Name string
	Num []int
}


func main(){
	var arr1 []int
	arr2 := make([]int,0)
	one := A{"one",arr1}
	two := A{"two",arr2}

	one1,err := json.Marshal(one)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(one1))

	one2,err := json.Marshal(two)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(one2))
}

----------
$ go run json.go
{"Name":"one","Num":null}
{"Name":"two","Num":[]}


```
在编码上，**==nil== slice 返回的是（json）null，而空 slice 返回的是空数组**。


## 6. 协程，线程，进程的区别。

>进程是系统资源分配的最小单位, 系统由一个个进程(程序)组成
一般情况下，包括文本区域（text region）、数据区域（data region）和堆栈（stack region）。
- 文本区域存储处理器执行的代码
- 数据区域存储变量和进程执行期间使用的动态分配的内存；
- 堆栈区域存储着活动过程调用的指令和本地变量。

通信问题:  由于进程间是隔离的,各自拥有自己的内存内存资源, 因此相对于线程比较安全, 所以不同进程之间的数据只能通过 IPC(Inter-Process Communication) 进行通信共享.

>线程:线程属于进程,共享进程的内存地址空间。同时多线程是不安全的,**当一个线程崩溃了,会导致整个进程也崩溃了,即其他线程也挂了,** 但**多进程而不会**,一个进程挂了,另一个进程依然照样运行。

进程切换分3步:

- 切换页目录以使用新的地址空间
- 切换内核栈
- 切换硬件上下文

而线程切换只需要第2、3步,因此进程的切换代价比较大

>协程:协程是属于线程的。协程程序是在线程里面跑的，因此协程又称微线程和纤程等**。协没有线程的上下文切换消耗**。协程的调度切换是用户(程序员)手动切换的,因此更加灵活,因此又叫**用户空间线程**.


## 7. 互斥锁，读写锁，死锁问题是怎么解决。
编码过程中加锁的地方记得释放锁。

`go tool vet`在编译阶段发现一些常规错误问题。
`go run -race`检测竞争态问题。


## 9. Data Race问题怎么解决？能不能不加锁解决这个问题？
- 避免对共享 variable 进行写操作
- 避免在多个 goroutine 中存取变量(使用channel)
- 使用锁来保护共享的 variable 来保证任意时刻只有一个 goroutine 对变量进行修改
- >`go run -race`检测竞争态问题。
能不能不加锁解决这个问题: 使用channel当作锁：`done <- struct{}{}`

## 10. 什么是channel，为什么它可以做到线程安全？

channel 的实现就是 队列 + 锁，源码分析可知。

11. goroutine 如何调度?

12. Golang GC 时会发生什么?

13. Golang 中 goroutine 的调度.

14. 并发编程概念是什么？

15. 负载均衡原理是什么

16. lvs相关

17. 微服务架构是什么样子的

## 18. 分布式锁实现原理，用过吗？
在分布式环境中某一个值或状态只能是唯一存在的，这是分布式锁实际的基本原理。
实现方式：

##### 基于数据库实现分布式锁

> 直接创建一张锁表，然后通过操作该表中的数据来实现了。当我们要锁住某个方法或资源时，我们就在该表中增加一条记录，想要释放锁的时候就删除这条记录。


###### 基于缓存实现分布式锁（redis，etcd）
>基于缓存来实现在性能方面会表现的更好一点。而且很多缓存是可以集群部署的，可以解决单点问题

###### 基于Zookeeper实现分布式锁
>每个客户端对某个方法加锁时，在zookeeper上的与该方法对应的指定节点的目录下，生成一个唯一的瞬时有序节点。 判断是否获取锁的方式很简单，只需要判断有序节点中序号最小的一个。 当释放锁的时候，只需将这个瞬时节点删除即可。同时，其可以避免服务宕机导致的锁无法释放，而产生的死锁问题。



## 19. etcd和redis怎么实现分布式锁


```
> set lock:codehole true ex 5 nx 
OK
... do something critical ...
> del lock:codehole
```

上面这个指令就是 setnx 和 expire 组合在一起的原子指令，它就是分布式锁的奥义所在。

#### 超时问题

Redis 的分布式锁不能解决超时问题，如果在加锁和释放锁之间的逻辑执行的太长，以至于超出了锁的超时限制，就会出现问题。

因为这时候第一个线程持有的锁过期了，临界区的逻辑还没有执行完，这个时候第二个线程就提前重新持有了这把锁，导致临界区代码不能得到严格的串行执行。

为了避免这个问题，**Redis分布式锁不要用于较长时间的任务。**

**更好一点的方案：为set指令的value参数设置为一个随机数，释放锁时先匹配随机数是否一致，然后再删除 key，这是为了确保当前线程占有的锁不会被其它线程释放，除非这个锁是过期了被服务器自动释放的**。

23. goroutine和channel的作用分别是什么

24. 怎么查看goroutine的数量

25. 说下go中的锁有哪些?
三种锁，读写锁，互斥锁，还有map的安全的锁

26. 读写锁或者互斥锁读的时候能写吗

27. 怎么限制goroutine的数量

## 28. channel是同步的还是异步的
同步的

## 29. 说一下异步和非阻塞的区别
>同步与异步:同步和异步关注的是消息通信机制 (synchronous communication/ asynchronous communication)

>阻塞与非阻塞 :阻塞和非阻塞关注的是程序在等待调用结果（消息，返回值）时的状态.

## 30. log包线程安全吗？
go 标准库的log包是线程安全的。

31. goroutine和线程的区别
 
32. 滑动窗口的概念以及应用

33. 怎么做弹性扩缩容，原理是什么

34. 让你设计一个web框架，你要怎么设计，说一下步骤

## 35. 说一下中间件原理
>类似于 Python 的装饰器，这归功于golang将函数作为一直引用类型，函数在go的类型系统里也是一等公民。而 go 的http中间件实现主要是利用高阶函数来设计。

**go 原生的http服务器主要依赖两种数据类型 ServeMux（map + 锁），http.Handler。http服务器处理路由与响应方法用 `http.Handle()`,`http.Handle`用`ServeMux`路由分发调用 `http.Handler`。**

一个go函数，或者对象方法，只要签名是 `func(ResponseWriter, *Request)` ，那么它的签名就与 `http.HandlerFunc` 一致，可以被强制转换为 `http.HandlerFunc`类型。同时`http.HandlerFunc`类型实现了`http.Handler`接口：


```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request){
    f(w, r)
}
```

所以我们就可以将我们的函数或者方法包装成 `http.Handler`,然后在返回一个新的Handler，这就是中间件的实现过程。



## 36. 怎么设计orm，让你写你要怎么写
1. 反射获得各个字段的名称和值
2. 拼接sql，执行sql
3. 判断是否有自增字段，如果有，则设置自增字段的值。

## 37. epoll原理

>**epoll是一种I/O事件通知机制**

#### I/O事件

**I/O**: 输入输出(input/output)，输入输出的对象可以是  文件(file),  网络(socket), 进程之间的管道(pipe),  在linux系统中，都用文件描述符(fd)来表示

**事件**:
- 可读事件，当文件描述符关联的内核读缓冲区可读，则触发可读事件什么是可读呢？就是内核缓冲区非空，有数据可以读取
- 可写事件,   当文件描述符关联的内核写缓冲区可写，则触发可写事件什么是可写呢？就是内核缓冲区不满，有空闲空间可以写入


#### 通知机制

通知机制，就是当事件发生的时候，去通知他
通知机制的反面，就是轮询机制

以上两点结合起来理解

>**epoll是一种当文件描述符的内核缓冲区非空的时候，发出可读信号进行通知，当写缓冲区不满的时候，发出可写信号通知的机制**

38. 用过原生的http包吗？

39. 一个非常大的数组，让其中两个数想加等于1000怎么算


```go
func twoSum(nums []int, target int) []int {
    var res []int
	co := make(map[int]int，len(nums))
	for i, v := range nums {
		co[v] = i
	}
	for i := 0; i < len(nums); i++ {
		t := target - nums[i]
		if value, ok := co[t]; ok && value != i {
			res = append(res, i)
			res = append(res, value)
			break
		}
	}
	return res
}
```


40. 各个系统出问题怎么监控报警

## 41. 常用测试工具，压测工具，方法

goconvey,vegeta

## 42. 复杂的单元测试怎么测试，比如有外部接口mysql接口的情况【goconvey】

其实很重要的一个部分就是被测代码本身是容易被测的，也就是说在设计和编写代码的时候就应该先想到相好如何单元测试，甚至有人提出**可以先写单元测试，再写具体被测代码。**因为一个接口(或者称为单元)在被设计好后，它实现就确定了，实际效果也确定了。

此外还可以通过Docker起一个mysql服务，运行create.sql和insert.sql,然后进行模拟测试。


43. redis集群，哨兵，持久化，事务

45. 高可用软件是什么

46. 怎么搞一个并发服务程序

47. 讲解一下你做过的项目，然后找问题问实现细节

48. mysql事务说一下

49. 怎么做一个自动化配置平台系统

50. grpc遵循什么协议

51. grpc内部原理是什么

## 52. http2的特点是什么，与http1.1的对比

    | HTTP1.1                    | HTTP2       | QUIC                        |
    | -------------------------- | ----------- | --------------------------- |
    | 持久连接                       | 二进制分帧       | 基于UDP的多路传输（单连接下）            |
    | 请求管道化                      | 多路复用（或连接共享） | 极低的等待时延（相比于TCP的三次握手）        |
    | 增加缓存处理（新的字段如cache-control） | 头部压缩        | QUIC为 传输层 协议 ，成为更多应用层的高性能选择 |
    | 增加Host字段、支持断点传输等（把文件分成几部分） | 服务器推送       |     