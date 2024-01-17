# channel
## 参考
[Draveness channel](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)

[Golang Concepts: Nil Channels](https://distributedsystemsmadeeasy.medium.com/golang-concepts-nil-channels-67b26a766a44)

[Go 程序员面试笔试宝典 通道](https://golang.design/go-questions/channel/csp/)

## 设计原理
`Do not communicate by sharing memory; instead, share memory by communicating.`

不要通过共享内存来通信，而要通过通信来实现内存共享。

## 数据结构
```go
type hchan struct {
	qcount   uint // 环形队列中的元素总数
	dataqsiz uint // 环形队列大小
	buf      unsafe.Pointer // 环形队列底层数组，只针对有缓存channel
	elemsize uint16 // chan元素大小
	closed   uint32 // chan是否关闭
	elemtype *_type // chan元素类型
	sendx    uint   // 环形队列中的发送索引
	recvx    uint   // 环形队列中的接收索引
	recvq    waitq  // 接收等待队列，双向链表
	sendq    waitq  // 发送等待队列，双向链表

	lock mutex // 保护hchan中所有字段
}
```
* 主要组成 = 环形队列 + 接收阻塞队列 + 发送阻塞队列

### 阻塞队列
阻塞队列 `runtime.waitq` 是遵循**先进先出**原则的双向链表，链表元素是将 goroutine 和待发送或待接收数据地址等进行封装后的 `runtime.sudog`

* sudog的必要性：g和并发对象是多对多关系，一个g可能在很多等待队列中。
```
sudog is necessary because the g ↔ synchronization object relation 
is many-to-many. A g can be on many wait lists, so there may be
many sudogs for one g; and many gs may be waiting on the same
synchronization object, so there may be many sudogs for one object.
```

## 创建
通过 `make` 函数创建，根据声明的元素类型和缓冲区大小选择不同的初始化策略：
* 无缓冲区：只分配 `runtime.hchan` 内存空间
* 有缓冲区：
    * channel中存储的不是指针类型：为 `runtime.hchan` 和 `hchan.buf`底层数组分配一块连续的内存空间
    * channel中存储的是指针类型：分别为 hchan 和 底层数组 分配内存空间

* channel是在**堆**上分配的，目的是为了goroutine之间的通信（因为每个G都有自己的栈）

## 发送
三种特判：
1. 如果 `hchan` 是 `nil`，那么会调用 `gopark` 挂起当前goroutine，直接返回。 
2. 非阻塞发送（select中）如果无缓冲区或者满了，那么直接返回。 
3. 如果channel已经关闭，panic。

三种情况：
1. 接收队列不为空
2. 缓冲区有空闲
3. 缓冲区无空闲

* 特判3和三种情况是要加锁的（`hchan.lock`）。

### 接收队列不为空
* 两种情况：无缓冲区channel、缓冲区为空的channel

取队头元素，获取sudog中保存的待接收数据地址，直接将发送的数据copy到待接收地址上，并且将sudog中保存的暂止状态的 goroutine 标记为 `Grunnable` 状态，放到P的 `runnext`位置上等待执行。解锁直接返回。

* 如何保证待接收数据的对象不被回收？
答：保存到等待队列前调用 `runtime.KeepAlive` 禁止被收集器回收。

#### 优点
降低了内存拷贝的性能损耗

暂止的goroutine不需要再获取channel锁来操作缓冲区，提高了速度

#### 缺点
一个运行的G直接对另一个G的**栈**进行了修改，事实上违反了GC中关于goroutine栈是各自独有的假设，因此需要在修改过程中加**写屏障**。

### 缓冲区有空间
获取 `hchan.sendx` 索引，将数据拷贝到缓冲区索引位置，更新索引。解锁直接返回。

### 缓冲区无空间
* 两种情况：无缓冲区channel、缓冲区已满channel

组装包含当前g以及相关信息的 sudog，将其入队，并调用 `gopark` 暂止当前的g，等待被唤醒。

## 接收
三个特判
1. 如果 `hchan` 是 `nil`，对于非阻塞接收直接返回；而对于阻塞接收会调用 `gopark` 挂起当前goroutine，直接返回。 
2. 非阻塞接收（select中）如果无缓冲区或者缓冲区为空，清除待接收指针对应的数据（将等号左边等待赋值的变量置为零值），直接返回。 
3. 如果channel已经关闭，并且缓冲区中没有数据，清除待接收指针对应的数据，直接返回。 

三种情况
1. 发送队列不为空
2. 缓冲区有数据
3. 缓冲区无数据

* 同样，特判3和三种情况是要加锁的（`hchan.lock`）。

### 发送队列不为空
如果是无缓冲区channel，那么直接将队头元素中保存的值copy给待接收变量。

否则，说明channel缓冲区已满，需要获取缓冲区中 `hchan.recvx` 索引对应的值，并取出发送队列头元素，将刚刚缓冲区索引对应位置的值用元素中保存的值代替，并更新 recvx 索引到下一个位置，唤醒这个暂止的发送goroutine，解锁直接返回。

简而言之，就从缓冲区取一个值并从发送队列递补。

### 缓冲区有数据
根据 `hchan.recvx` 索引从缓冲区取一个值赋给等号左边待接收变量，更新索引，解锁直接返回。

### 缓冲区无数据
组装包含当前g以及相关信息的 sudog，将其入队，并调用 `gopark` 暂止当前的g，等待被唤醒。

## 关闭
当关闭一个 nil channel 或者关闭一个已经关闭的 channel 时，panic。

将接收和发送队列中所有等待中的goroutine取出来唤醒。其中发送队列的所有goroutine都会panic。

## nil channel 有什么用？
可以用来动态地控制哪些 case 可以在 select 中被执行。通过将 channel 设置为 nil 临时禁用通道交互。具体见[select关键字](https://github.com/sweetpear0108/learning-note/tree/main/golang/select.md)

## channel 引发的资源泄露
对于一个channel，如果没有任何的goroutine引用，那么会进行GC将其回收。

而出现资源泄露是因为channel处于满或空的状态，一直得不到改变，发送或接收的goroutine一直在等待队列得不到释放。

## 应用
* 与 Timer 结合可以实现超时控制和定时任务
* 解耦生产方消费方，异步起几个 worker 通过 channel 读取任务
* 通过缓冲区大小限制并发数

## runtime/chan.go 出现的内存对齐操作

```
maxAlign  = 8
hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
```

**作用**：对齐内存，低3位保证为0，即hchanSize为8的倍数

maxAlign-1为掩码，-int(unsafe.Sizeof(hchan{}))为补码，&运算只保留低3位bit，这样在hchanSize低3位上总为0

**例子**
| 表达式                                              | 例1   | 例2   |
| :----                                              | :---- | :---- |
| unsafe.Sizeof(hchan{})                             | 01101 | 00101 |
| -int(unsafe.Sizeof(hchan{}))                       | 10011 | 11011 |
| maxAlign-1                                         | 00111 | 00111 |
| uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1)) | 00011 | 00011 |
| hchanSize                                          | 10000 | 01000 |
