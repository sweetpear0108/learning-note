# I/O模型和网络轮询器netpoll
## 参考
[Java并发之BIO NIO AIO IO多路复用的区别](https://blog.csdn.net/u014453898/article/details/109811000)

[网络轮询器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/)

[小林coding](https://www.xiaolincoding.com/os/8_network_system/selete_poll_epoll.html#epoll)

[面试官：请你谈谈关于IO同步、异步、阻塞、非阻塞的区别？](https://blog.csdn.net/pisa8559/article/details/130962402)

[潘少：Go netpoller 原生网络模型之源码全面揭秘](https://taohuawu.club/archives/go-netpoll-io-multiplexing-reactor#toc-head-0)

## 五种I/O模型
同步阻塞 I/O（BIO）、同步非阻塞 I/O（NIO）、信号驱动 I/O（SIGIO）、异步非阻塞 I/O（AIO） 、I/O 多路复用（I/O multiplexing）

* I/O两阶段：等待数据准备、将数据从内核（用户程序）拷贝到用户程序（内核），见下图；

* 阻塞/非阻塞：主要看第一步，线程在`资源就绪`之前是否需要等待
* 同步/异步：主要看第二步，请求线程在`数据用户态内核态之间的拷贝`期间是否被阻塞

![五种](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*cmTCOxq2HXwJNBgL.png)

* linux中文件描述符是什么：文件描述符（file descriptor）就是内核为了高效管理这些已经被打开的文件所创建的索引，其是一个非负整数（通常是小整数），用于指代被打开的文件，所有执行I/O操作的系统调用都通过文件描述符来实现。

### 同步阻塞I/O（BIO）
最常见的IO模式，例如读写文件时执行read write系统调用。

从开始执行系统调用开始，用户线程就会进入阻塞状态直到I/O操作结束。

* 优点：简单，线程等待期间基本不会占用CPU资源
* 缺点：高并发下大量线程被阻塞，导致内存和线程切换开销巨大

### 同步非阻塞I/O（NIO）
用户程序会不断轮询发起系统调用，直到内核中资源准备就绪，再完成I/O操作。

* 优点：可以在等待过程中执行其他任务，提高 CPU 利用率
* 缺点：需要不断轮询占用大量 CPU 时间，且轮询间隔可能会放大响应延迟，降低整体数据吞吐量

### 异步非阻塞I/O（AIO）
用户线程只需要负责向内核发送一个系统调用请求，等待数据就绪和数据拷贝的过程都由内核来完成，完成后内核通过信号或回调函数通知用户线程I/O操作完成。

* 优点：减少了 NIO 中轮询占用的 CPU 资源，且减少了响应延迟
* 缺点：复杂，回调函数占用系统资源，使用信号通信多平台可移植性差

### 多路复用
**事件驱动**机制，使用特定的系统调用（select、poll、epoll等）**阻塞**地监听一组文件描述符，当文件描述符就绪（状态转变为可读或者可写）时，通知程序继续进行相应的I/O操作。

多路复用也属于同步策略，与NIO比较类似，只不过是由特定的系统调用负责去轮询资源是否就绪，而不是用户线程亲力亲为

为什么属于同步策略：特定系统调用只负责告诉用户程序资源是否就绪，而（在read调用中）数据从内核复制到用户内存的操作还是由用户程序发送系统调用触发

为什么可以处理大量连接：NIO一个请求需要一个线程去处理，而多路复用则是做到了**多路**请求**复用**一个线程，大大减少了资源占用

* 优点：可以同时处理大量连接，减小系统开销，响应性能好
* 缺点：select和poll都使用**线性结构**（数组和链表）存储文件描述符，需要拷贝到内核内存开销大，返回就绪事件需要遍历时间复杂度高，select还有数组大小限制

此外，由于特定系统调用是轮询的并且拷贝数据的时候用户线程也是阻塞的，所以多路复用只有在处理高并发请求的时候才能体现性能优势。

#### epoll
epoll解决了select、poll存在的内存开销和时间复杂度问题

* epoll使用**红黑树**的数据结构追踪所有待检测的文件描述符，每次只需要加入一个待检测的文件描述符，而不像select、poll一样每次操作都穿入整个集合，加入时间复杂度O(logn)

* 维护wait**链表**来记录就绪事件，而不是遍历整个集合

![epoll](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8/epoll.png)

#### 事件驱动
* 边缘触发：当被监控的文件描述符上第一次出现事件就绪时，通知用户程序
* 水平触发：当被监控的文件描述符上存在就绪事件，通知用户程序

## go中的网络轮询器netpoll
netpoll的实现原理是**多路复用+NIO**（见最后[总结](#总结)部分），go中使用原生网络模型构建tcp server时，网络模型底层就是基于netpoll实现的

go中使用 `runtime.pollDesc` 封装操作系统的文件描述符，空闲描述符以链表形式存储。

五个函数
```go
func netpollinit() // 创建实例
func netpollopen(fd uintptr, pd *pollDesc) int32 // 注册fd
func netpoll(delta int64) gList // 等待事件操作
func netpollBreak() // 停止epoll
func netpollIsPollDescriptor(fd uintptr) bool // 判断fd是否被netpoll使用
```

go中针对不同平台采用了不同版本的网络轮询模块，linux-epoll，darwin-kqueue...下面以epoll为例。
### 初始化
`netpollinit` 函数只会在初始化网络IO或文件IO或计时器时**调用一次**（sync.Once来实现），其作用是：
1. 创建一个新的epoll文件描述符
2. 创建一个用于通信的管道
3. 将 `netpollBreakRd` 通知信号量封装成 `epollevent` 事件结构体注册进 epoll 实例。

* netpollBreakRd 是一个用于传递控制信号的管道，监听这个事件的目的是为了实现网络轮询的终止。

然后，会批量初始化pollDesc并调用 `netpollopen` 函数注册轮询事件监听文件描述符（如net包中的listener fd）到epoll实例

### 等待事件
当在文件描述符上执行读写操作时（如net包中对请求Read或Write），会调用`runtime.poll_runtime_pollWait`：
1. 检查描述符状态，如果fd不可读/写，则会执行`gopark`让出当前线程，将Goroutine转换到休眠状态。
2. 将goroutine保存在`pollDesc`中（pollDesc是go封装的文件描述符），将`pollDesc`加入epoll底层基于红黑树的阻塞队列`eventpoll.rbr`中。
3. 等待描述符变为可读或可写之后通知运行时，唤醒休眠的Goroutine。

### 轮询等待
调用`netpoll`函数获取需要唤醒的goroutines：
1. 根据传入的参数确定epoll系统调用需要等待的时间（永久阻塞、非阻塞、有限期阻塞）。
2. 通过`epollwait` 系统调用获取待处理事件的数量。
3. 根据返回的待处理事件数量将`eveentpoll.rdllist`就绪链表中的额所有`pollDesc`中存储的Goroutine加入运行队列（本地或全局）等待调度器的调度。

### 原生网络库
调度过程
![net](https://res.strikefreedom.top/static_res/blog/figures/netpoller-goroutines-scheduling.png)

## netpoll的价值
前置知识：
1. 当一个运行的 G 发生系统调用时，不只是 G 会被阻塞，执行该 G 的 M 也会被阻塞，
2. go并发的优势就在于将任务的执行载体细化到了用户级的 G

不使用netpoll会存在的问题：
* 对于I/O密集型任务，大量系统调用会导致 M 频繁的挂起、新建、唤醒，背离了golang运行时调度的设计思想，即程序的控制权掌握在用户进程而不是内核，导致无法使用强大的调度器进行并发调度了

netpoll的价值：
* 发生系统调用时，运行时将 G park住并将包装后的描述符加入epoll等待就绪，避免了因为陷入内核态导致 M 被阻塞，因此这个 M 还可以继续配合调度器运行新的 G。

## 总结
### 为什么说 netpoll 是 NIO + 多路复用？
NIO指的是对于**线程 M**，发生I/O操作时不会因为系统调用而被阻塞

多路复用指的是操作系统层面，使用提供的 epoll、kqueue等多路复用技术

### netpoll优势
保证了在I/O密集型服务中，go依旧可以实现在运行中进行用户级调度，提高了并发能力，减少了系统资源占用

### netpoll存在的问题
服务器在短时间内建立了海量连接，会导致创建大量的goroutine，过多的消耗系统资源。

解决办法：Reactor模型