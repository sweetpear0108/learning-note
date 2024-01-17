# select 关键字

## 参考
[draveness select关键字](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/#52-select)

## 设计思想
与操作系统中多路复用模型相似，同时监听多个 channel 状态，等待变为可读或可写。

与 `switch` 关键字的控制结构相似，只不过 `select` 的 `case` 表达式**必须是** channel 的收发操作。

select 两个特别之处：
1. 在 channel 上可以进行非阻塞的收发操作
2. 在多个 case 同时满足收发状态时，随机执行其中之一

### 非阻塞收发
当 select 中存在 `default` 分支时，只需要知道其他分支中的 channel 收发状态是否就绪，如果全部未就绪就会执行 `default` 分支上的操作。

因此，在其他分支上对于 channel 的读写操作都是**非阻塞**的，具体实现见[channel](https://github.com/sweetpear0108/learning-note/tree/main/golang/channel.md)这一部分。

### 随机执行
在 `runtime.selectgo` 开头，通过 `runtime.fastrandn` 函数随机打乱case的轮询顺序 `pollOrder`。

目的：避免饥饿问题的发生

## 实现原理
每个case对应一个 `runtime.scase` 结构体，结构体中保存着收发操作目标 channel 和接收到的数据要放到的内存地址（等号左边变量指针）。

### 编译期间

在**编译**过程中，会根据 case 的不同情况对控制语句进行优化：

1. 不存在任何case：调用 `gopark` 永久阻塞当前 goroutine。

2. 只存在一个case：改写为 `if` 条件语句判断是否为 `nil channel`，是则永久阻塞当前 goroutine，否则正常执行 channel 的收发操作。 

3. 一个普通case，一个default：改写为 `if-else`条件语句，if 中调用**非阻塞**channel收发函数，当返回的状态为未就绪时，执行 else 中的逻辑。

4. 多个case：将所有case转化为 `runtime.scase` 结构体数组，调用 `runtime.selectgo` 函数确定可执行的case对应的结构体。

### 运行时期间

运行时主要就是执行 `runtime.selectgo` 这个函数，目的是确定可执行的scase结构体。

主要步骤：
1. 初始化两顺序：随机打乱轮询顺序（避免饥饿），按照channel地址排序确定加锁顺序（避免死锁）。如果 case 中 channel 为 nil，则在后续的轮询中**忽略**这个case（不加入顺序切片）。
2. 给 select 中的所有 channel 加锁。
3. 按轮询顺序**遍历**所有 case 查找存在的准备就绪的 channel，如果有，立刻执行，解锁全部channel后返回。
4. 对于非阻塞操作（存在 default），直接全部解锁并返回。
5. 按加锁顺序**遍历**所有 case，将当前 goroutine 包装为 sudog，加入每一个channel的等待队列中，此外还会附着在当前 goroutine 的 waiting 链表末尾。
6. 调用 gopark 暂止当前 goroutine，解锁所有 channel，等待有就绪事件时被唤醒。
7. 给 select 中的所有 channel 加锁。
8. 按加锁顺序**遍历**所有 case，找到就绪的 case，将其他case中没有被用到的 sudog 从 channel 等待队列出队并释放掉。
9. 解锁所有 channel，返回选中的case的索引，从而可以执行对应的逻辑。


#### 不按顺序加锁为什么会导致死锁？
在并发加锁的时候会出现问题。例如：两个goroutine中分别有两个select，g1按顺序对 chan1、chan2 加锁，g2按顺序对 chan2、chan1 加锁。

由于select需要将内部所有channel都加锁才会执行任务，当按g1-chan1、g2-chan2顺序加锁时，g1-chan2、g2-chan1会阻塞，导致死锁。


#### nil channel 的用处与原理
在上述步骤1中，确定 case 轮询顺序时不会包括channel为nil的case，从而在后续的遍历过程中不会涉及这种case，因此可以说 nil channel 的 case 被永久阻塞了。

将某个case的channel置为nil可以禁用当前分支，而且其他的分支不会受影响。这样做可以在```for...select...```这种结构中避免一个已经完成所有任务的channel反复被检查，节约了系统资源。