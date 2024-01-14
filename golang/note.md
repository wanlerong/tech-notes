# reference

主要：
- [The Go Programming Language Specification](https://go.dev/ref/spec)

次要：
- [Go 程序员面试笔试宝典](https://golang.design/go-questions/)
- [Go 语言设计与实践](https://draveness.me/golang/)

# 基础原理


# 调度器

goroutine 和线程的区别

操作系统对goroutine是无感知的，操作系统执行的依然是线程/进程。是 Runtime 维护所有的 goroutines，并通过 scheduler 来进行调度。


内存占用：创建一个 goroutine 的栈内存消耗为 2 KB，实际运行过程中，如果栈空间不够用，会自动进行扩容。创建一个 thread 则需要消耗 1 MB 栈内存。

创建和销毀：Thread 创建和销毀都会有巨大的消耗，因为要和操作系统打交道，是内核级的，通常解决的办法就是线程池。而 goroutine 因为是由 Go runtime 负责管理的，创建和销毁的消耗非常小，是用户级。

切换：
goroutines 切换消耗小。当 threads 切换时，需要保存各种寄存器，以便将来恢复。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-164626.png)

### GMP

#### G - Goroutine，Go协程，是参与调度与执行的最小单位。

它包含：表示 goroutine 栈的一些字段，指示当前 goroutine 的状态，指示当前运行到的指令地址，也就是 PC 值。
主要保存 goroutine 的一些状态信息以及 CPU 的一些寄存器的值，例如 IP 寄存器，以便在轮到本 goroutine 执行时，CPU 知道要从哪一条指令处开始执行。

当 goroutine 被调离 CPU 时，调度器负责把 CPU 寄存器的值保存在 g 对象的成员变量之中。
当 goroutine 被调度起来运行时，调度器又负责把 g 对象的成员变量所保存的寄存器值恢复到 CPU 的寄存器。

// 描述栈的数据结构，栈的范围：[lo, hi)
type stack struct {
    // 栈顶，低地址
	lo uintptr
	// 栈低，高地址
	hi uintptr
}

还包括 PC，SP 等寄存器。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-180643.png)


#### M - Machine，指的是系统线程，内核线程，包含正在运行的 goroutine 等字段

M 是真正工作的人。结构体 m 就是我们常说的 M，它保存了 M 自身使用的栈信息、当前正在 M 上执行的 G 信息、与之绑定的 P 信息……

当 M 没有工作可做的时候，在它休眠前，会“自旋”地来找工作：检查全局队列，查看 network poller，试图执行 gc 任务，或者“偷”工作。

M 会从与它绑定的 P 的本地队列获取可运行的 G，也会从 network poller 里获取可运行的 G，还会从其他 P 偷 G。

M 只有自旋和非自旋两种状态。自旋的时候，会努力找工作；找不到的时候会进入非自旋状态，之后会休眠，直到有工作需要处理时，被其他工作线程唤醒，又进入自旋状态。


#### P - Processor，代表一个虚拟的 Processor。它维护一个处于 Runnable 状态的 g 队列，m 需要获得 p 才能运行 g。

为 M 的执行提供“上下文”，保存 M 执行 G 时的一些资源，例如本地可运行 G 队列，memeory cache 等。
一个 M 只有绑定 P 才能执行 goroutine，当 M 被阻塞时，整个 P 会被传递给其他 M ，或者说整个 P 被接管。
可以实现 M 之间的负载平衡。
![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-180744.png)


当然，在 Go 的早期版本，并没有 p 这个结构体，m 必须从一个全局的队列里获取要运行的 g，因此需要获取一个全局的锁，当并发量大的时候，锁就成了瓶颈。后来在大神 Dmitry Vyokov 的实现里，加上了 p 结构体。每个 p 自己维护一个处于 Runnable 状态的 g 的队列，解决了原来的全局锁问题。


如何按OS类比的话：感觉 g 可类比为线程，m 就是cpu，p 是每个cpu的上下文信息，比如各自的就绪队列。



Runtime 起始时会启动一些 G：垃圾回收的 G，执行调度的 G，运行用户代码的 G；并且会创建一个 M 用来开始 G 的运行。随着时间的推移，更多的 G 会被创建出来，更多的 M 也会被创建出来。


Go scheduler 的目标：

For scheduling goroutines onto kernel threads.


- use a small number kernel threads
	ideas:reuse threads   limit the number of goroutine-running threads 

- Supporthigh concurrency.
   ideas: threads use independent runqueues & keep them balanced
 
- leverage parallelism i.e.scale to N cores.
  ideas: use a runqueue per core & employ thread spinning.


线程私有的 runqueues, 并且可以从其他线程 stealing goroutine 来运行，线程阻塞后，可以将 runqueues 传递给其他线程。

当一个线程阻塞的时候，将和它绑定的 P 上的 goroutines 转移到其他线程。

Go scheduler 会启动一个后台线程 sysmon，用来检测长时间（超过 10 ms）运行的 goroutine，将其调度到 global runqueues。这是一个全局的 runqueue，优先级比较低，以示惩罚。


2核的CPU， 超线程机制，1 个核可以变成 2 个
因为 NumCPU 返回的是逻辑核心数，而非物理核心数，所以最终结果是 4。

Go 程序启动后，会给每个逻辑核心分配一个 P（Logical Processor）；同时，会给每个 P 分配一个 M（Machine，表示内核线程），这些内核线程仍然由 OS scheduler 来调度。

总结一下，当我在本地启动一个 Go 程序时，会得到 4 个系统线程去执行任务，每个线程会搭配一个 P。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-173333.png)


全局可运行队列（GRQ）和本地可运行队列（LRQ）。 LRQ 存储本地（也就是具体的 P）的可运行 goroutine，GRQ 存储全局的可运行 goroutine，这些 goroutine 还没有分配到具体的 P。


Go scheduler 是 Go runtime 的一部分，它内嵌在 Go 程序里，和 Go 程序一起运行。因此它运行在用户空间，在 kernel 的上一层。和 Os scheduler 抢占式调度（preemptive）不一样，Go scheduler 采用协作式调度（cooperating）。

goroutine 的状态
- Waiting	等待状态，goroutine 在等待某件事的发生。例如等待网络数据、硬盘；调用操作系统 API；等待内存同步访问条件 ready，如 atomic, mutexes
- Runnable	就绪状态，只要给 M 我就可以运行
- Executing	运行状态。goroutine 在 M 上执行指令，这是我们想要的

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-174321.png)



### 调度时机

- 使用关键字 go	go 创建一个新的 goroutine，Go scheduler 会考虑调度
- GC	由于进行 GC 的 goroutine 也需要在 M 上运行，因此肯定会发生调度。当然，Go scheduler 还会做很多其他的调度，例如调度不涉及堆访问的 goroutine 来运行。GC 不管栈上的内存，只会回收堆上的内存
- 系统调用	当 goroutine 进行系统调用时，会阻塞 M，所以它会被调度走，同时一个新的 goroutine 会被调度上来
- 内存同步访问	 atomic，mutex，channel 操作等会使 goroutine 阻塞，因此会被调度走。等条件满足后（例如其他 goroutine 解锁了）还会被调度上来继续运行


### task steal

Go scheduler 的职责就是将所有处于 runnable 的 goroutines 均匀分布到在 P 上运行的 M。

当一个 P 发现自己的 LRQ 已经没有 G 时，会从其他 P “偷” 一些 G 来运行。看看这是什么精神！自己的工作做完了，为了全局的利益，主动为别人分担。这被称为 Work-stealing，Go 从 1.1 开始实现。

如果 P 上的 M 阻塞了，那它就需要其他的 M 来运行 P 的 LRQ 里的 goroutines。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-175114.png)



runtime.schedule() {
    // only 1/61 of the time, check the global runnable queue for a G.
    // if not found, check the local queue.
    // if not found,
    //     try to steal from other Ps.
    //     if not, check the global runnable queue.
    //     if not found, poll network.
}

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-175509.png)



### 初始化

Go scheduler 在源码中的结构体为 schedt，保存调度器的状态信息、全局的可运行 G 队列、空闲的 M 链表，空闲的 M 数量等

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240112-181711.png)



# 垃圾回收

垃圾回收器的执行过程被划分为两个半独立的组件：

赋值器（Mutator）：这一名称本质上是在指代用户态的代码。因为对垃圾回收器而言，用户态的代码仅仅只是在修改对象之间的引用关系，也就是在对象图（对象之间引用关系的一个有向图）上进行操作。
回收器（Collector）：负责执行垃圾回收的代码。


所有的 GC 算法其存在形式可以归结为追踪（Tracing）和引用计数（Reference Counting）这两种形式的混合运用。

## 引用计数式 GC

每个对象自身包含一个被引用的计数器，当计数器归零时自动得到回收。因为此方法缺陷较多，在追求高性能时通常不被应用。Python、Objective-C 等均为引用计数式 GC。


## 追踪式 GC

从根对象出发，根据对象之间的引用信息，一步步推进直到扫描完毕整个堆并确定需要保留的对象，从而回收所有可回收的对象。Go、 Java、V8 对 JavaScript 的实现等均为追踪式 GC。
根对象:全局变量, 执行栈(每个 goroutine 都包含自己的执行栈), 寄存器.

追踪式，分为多种不同类型，例如：
- 标记清扫：从根对象出发，将确定存活的对象进行标记，并清扫可以回收的对象。
- 标记整理：为了解决内存碎片问题而提出，在标记过程中，将对象尽可能整理到一块连续的内存上。
- 增量式：将标记与清扫的过程分批执行，每次执行很小的部分，从而增量的推进垃圾回收，达到近似实时、几乎无停顿的目的。
- 增量整理：在增量式的基础上，增加对对象的整理过程。
- 分代式：将对象根据存活时间的长短进行分类，存活时间小于某个值的为年轻代，存活时间大于某个值的为老年代，永远不会参与回收的对象为永久代。并根据分代假设（如果一个对象存活时间不长则倾向于被回收，如果一个对象已经存活很长时间则倾向于存活更长时间）对对象进行回收。


对于 Go 而言，Go 的 GC 目前使用的是无分代（对象没有代际之分）、不整理（回收过程中不对对象进行移动与整理）、并发（与用户代码并发执行）的三色标记清扫算法。

不整理的原因：但 Go 运行时的分配算法基于 tcmalloc，基本上没有碎片问题。基于 tcmalloc 的现代内存分配算法，对对象进行整理不会带来实质性的性能提升。

无分代的原因：分代 GC 依赖分代假设，即 GC 将主要的回收目标放在新创建的对象上（存活时间短，更倾向于被回收），而非频繁检查所有对象。但 Go 的编译器会通过逃逸分析将大部分新生对象存储在栈上（栈直接被回收），只有那些需要长期存在的对象才会被分配到需要进行垃圾回收的堆中。也就是说，分代 GC 回收的那些存活时间短的对象在 Go 中是直接被分配到栈上，当 goroutine 死亡后栈也会被直接回收，不需要 GC 的参与，进而分代假设并没有带来直接优势。并且 Go 的垃圾回收器与用户代码并发执行，使得 STW 的时间与对象的代际、对象的 size 没有关系。Go 团队更关注于如何更好地让 GC 与用户代码并发执行（使用适当的 CPU 来执行垃圾回收），而非减少停顿时间这一单一目标上。

## 三色标记法

从垃圾回收器的视角来看，三色抽象规定了三种不同类型的对象，并用不同的颜色相称：

白色对象（可能死亡）：未被回收器访问到的对象。在回收开始阶段，所有对象均为白色，当回收结束后，白色对象均不可达。
灰色对象（波面）：已被回收器访问到的对象，但回收器需要对其中的一个或多个指针进行扫描，因为他们可能还指向白色对象。
黑色对象（确定存活）：已被回收器访问到的对象，其中所有字段都已被扫描，黑色对象中任何一个指针都不可能直接指向白色对象。

这样三种不变性所定义的回收过程其实是一个波面不断前进的过程，这个波面同时也是黑色对象和白色对象的边界，灰色对象就是这个波面。

当垃圾回收开始时，只有白色对象。随着标记过程开始进行时，灰色对象开始出现（着色），这时候波面便开始扩大。当一个对象的所有子节点均完成扫描时，会被着色为黑色。当整个堆遍历完成时，只剩下黑色和白色对象，这时的黑色对象为可达对象，即存活；而白色对象为不可达对象，即死亡。这个过程可以视为以灰色对象为波面，将黑色对象和白色对象分离，使波面不断向前推进，直到所有可达的灰色对象都变为黑色对象为止的过程。如下图所示：

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240113-153906.png)


## 如何观察 Go GC？

STW 可以是 Stop the World 的缩写，在这个过程中整个用户代码被停止或者放缓执行， STW 越长，对用户代码造成的影响（例如延迟）就越大。STW 如今已经优化到了半毫秒级别以下


### 环境变量：GODEBUG=gctrace=1 

可看执行日志：

对于用户代码向运行时申请内存产生的垃圾回收：
gc 2 @0.001s 2%: 0.018+1.1+0.029 ms clock, 0.22+0.047/0.074/0.048+0.34 ms cpu, 4->7->3 MB, 5 MB goal, 12 P

字段	含义
gc 2	第二个 GC 周期
0.001	程序开始后的 0.001 秒
2%	该 GC 周期中 CPU 的使用率
0.018	标记开始时， STW 所花费的时间（wall clock）
1.1	标记过程中，并发标记所花费的时间（wall clock）
0.029	标记终止时， STW 所花费的时间（wall clock）
0.22	标记开始时， STW 所花费的时间（cpu time）
0.047	标记过程中，标记辅助所花费的时间（cpu time）
0.074	标记过程中，并发标记所花费的时间（cpu time）
0.048	标记过程中，GC 空闲的时间（cpu time）
0.34	标记终止时， STW 所花费的时间（cpu time）
4	标记开始时，堆的大小的实际值
7	标记结束时，堆的大小的实际值
3	标记结束时，标记为存活的对象大小
5	标记结束时，堆的大小的预测值
12	P 的数量


wall clock 是指开始执行到完成所经历的实际时间，包括其他程序和本程序所消耗的时间； cpu time 是指特定程序使用 CPU 的时间； 他们存在以下关系：

wall clock < cpu time: 充分利用多核
wall clock ≈ cpu time: 未并行执行
wall clock > cpu time: 多核优势不明显


对于运行时向操作系统申请内存产生的垃圾回收（向操作系统归还多余的内存）：

scvg: 8 KB released
scvg: inuse: 3, idle: 60, sys: 63, released: 57, consumed: 6 (MB)
含义由下表所示：

字段	含义
8 KB released	向操作系统归还了 8 KB 内存
3	已经分配给用户代码、正在使用的总内存大小 (MB)
60	空闲以及等待归还给操作系统的总内存大小（MB）
63	通知操作系统中保留的内存大小（MB）
57	已经归还给操作系统的（或者说还未正式申请）的内存大小（MB）
6	已经从操作系统中申请的内存大小（MB）

### go tool trace 
go tool trace 的主要功能是将统计而来的信息以一种可视化的方式展示给用户。

package main

runtime/trace 包

func main() {
	f, _ := os.Create("trace.out")
	defer f.Close()
	trace.Start(f)
	defer trace.Stop()
	(...)
}

go tool trace trace.out



### 方式3：debug.ReadGCStats

此方式可以通过代码的方式来直接实现对感兴趣指标的监控，例如我们希望每隔一秒钟监控一次 GC 的状态：

```
func printGCStats() {
	t := time.NewTicker(time.Second)
	s := debug.GCStats{}
	for {
		select {
		case <-t.C:
			debug.ReadGCStats(&s)
			fmt.Printf("gc %d last@%v, PauseTotal %v\n", s.NumGC, s.LastGC, s.PauseTotal)
		}
	}
}
func main() {
	go printGCStats()
	(...)
}
```

### runtime.ReadMemStats

```
func printMemStats() {
	t := time.NewTicker(time.Second)
	s := runtime.MemStats{}

	for {
		select {
		case <-t.C:
			runtime.ReadMemStats(&s)
			fmt.Printf("gc %d last@%v, next_heap_size@%vMB\n", s.NumGC, time.Unix(int64(time.Duration(s.LastGC).Seconds()), 0), s.NextGC/(1<<20))
		}
	}
}
func main() {
	go printMemStats()
	(...)
}
```


## 内存泄露

预期的能很快被释放的内存由于附着在了长期存活的内存上、或生命期意外地被延长，导致预计能够立即回收的内存而长时间得不到回收。


### 形式1：预期能被快速释放的内存因被根对象引用而没有得到迅速释放

当有一个全局对象时，可能不经意间将某个变量附着在其上，且忽略的将其进行释放，则该内存永远不会得到释放。例如：

var cache = map[interface{}]interface{}{}

func keepalloc() {
	for i := 0; i < 10000; i++ {
		m := make([]byte, 1<<10)
		cache[i] = m
	}
}

### 形式2：goroutine 泄漏

Goroutine 作为一种逻辑上理解的轻量级线程，需要维护执行用户代码的上下文信息。在运行过程中也需要消耗一定的内存来保存这类信息，而这些内存在目前版本的 Go 中是不会被释放的。因此，如果一个程序持续不断地产生新的 goroutine、且不结束已经创建的 goroutine 并复用这部分内存，就会造成内存泄漏的现象，例如：

func keepalloc2() {
	for i := 0; i < 100000; i++ {
		go func() {
			select {}
		}()
	}
}

channel 的泄漏本质上与 goroutine 泄漏存在直接联系。Channel 作为一种同步原语，会连接两个不同的 goroutine，如果一个 goroutine 尝试向一个没有接收方的无缓冲 channel 发送消息，则该 goroutine 会被永久的休眠，整个 goroutine 及其执行栈都得不到释放，例如：

```
var ch = make(chan struct{})

func keepalloc3() {
	for i := 0; i < 100000; i++ {
		// 没有接收方，goroutine 会一直阻塞
		go func() { ch <- struct{}{} }()
	}
}
```

## 并发标记清除法的难点是什么？

在没有用户态代码并发修改三色抽象的情况下，回收可以正常结束。但是并发回收的根本问题在于，用户态代码在回收过程中会并发地更新对象图，从而造成赋值器和回收器可能对对象图的结构产生不同的认知。

不应出现对象的丢失，也不应错误的回收还不需要回收的对象。

可以证明，当以下两个条件同时满足时会破坏垃圾回收器的正确性：

条件 1: 赋值器修改对象图，导致某一黑色对象引用白色对象；
条件 2: 从灰色对象出发，到达白色对象的、未经访问过的路径被赋值器破坏。
只要能够避免其中任何一个条件，则不会出现对象丢失的情况，因为：

如果条件 1 被避免，则所有白色对象均被灰色对象引用，没有白色对象会被遗漏；
如果条件 2 被避免，即便白色对象的指针被写入到黑色对象中，但从灰色对象出发，总存在一条没有访问过的路径，从而找到到达白色对象的路径，白色对象最终不会被遗漏。

Dijkstra 插入屏障

为了防止黑色对象指向白色对象，会先把白色对象变灰，再指向黑色对象。避免了条件1

缺点：在标记阶段中，每次进行指针赋值操作时，都需要引入写屏障，这无疑会增加大量性能开销；为了避免造成性能问题，Go 团队在最终实现时，没有为所有栈上的指针写操作，启用写屏障，而是当发生栈上的写操作时，将栈标记为灰色，但此举产生了灰色赋值器，将会需要标记终止阶段 STW 时对这些栈进行重新扫描。

Yuasa 删除屏障
其基本思想是避免满足条件 2：防止丢失从灰色对象到白色对象的路径，通过创造另一条路径。
会先把黑色对象变灰
缺点： Yuasa 删除屏障会拦截写操作，进而导致波面的退后，产生“冗余”的扫描

## 垃圾回收的实现

阶段	说明	赋值器状态
SweepTermination	清扫终止阶段，为下一个阶段的并发标记做准备工作，启动写屏障	STW
Mark	扫描标记阶段，与赋值器并发执行，写屏障开启	并发
MarkTermination	标记终止阶段，保证一个周期内标记任务完成，停止写屏障	STW
GCoff	内存清扫阶段，将需要回收的内存归还到堆中，写屏障关闭	并发
GCoff	内存归还阶段，将过多的内存归还给操作系统，写屏障关闭	并发

Go 语言中对 GC 的触发时机存在两种形式：

主动触发，通过调用 runtime.GC 来触发 GC，此调用阻塞式地等待当前 GC 运行完毕。

被动触发，分为两种方式：
- 使用系统监控，当超过两分钟没有产生任何 GC 时，强制触发 GC。
- 使用步调（Pacing）算法，其核心思想是控制内存增长的比例。

除此之外，步调算法还需要考虑 CPU 利用率的问题，显然我们不应该让垃圾回收器占用过多的 CPU，即不应该让每个负责执行用户 goroutine 的线程都在执行标记过程。理想情况下，在用户代码满载的时候，GC 的 CPU 使用率不应该超过 25%，

通过 GOGC 或者 debug.SetGCPercent 进行控制（他们控制的是同一个变量，即堆的增长率 ρ）。整个算法的设计考虑的是优化问题：如果设上一次 GC 完成时，内存的数量为 Hm（heap marked），估计需要触发 GC 时的堆大小 HT（heap trigger），使得完成 GC 时候的目标堆大小 Hg（heap goal） 与实际完成时候的堆大小 Ha（heap actual）最为接近，即：

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240113-165657.png)


如果内存分配速度超过了标记清除的速度怎么办？ 
目前的 Go 实现中，当 GC 触发后，会首先进入并发标记的阶段。并发标记会设置一个标志，并在 mallocgc 调用时进行检查。当存在新的内存分配时，会暂停分配内存过快的那些 goroutine，并将其转去执行一些辅助标记（Mark Assist）的工作，从而达到放缓继续分配、辅助 GC 的标记工作的目的。


## go gc 如何调优

CPU 利用率：回收算法会在多大程度上拖慢程序？有时候，这个是通过回收占用的 CPU 时间与其它 CPU 时间的百分比来描述的。
GC 停顿时间：回收器会造成多长时间的停顿？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿。
GC 停顿频率：回收器造成的停顿频率是怎样的？目前的 GC 中需要考虑 STW 和 Mark Assist 两个部分可能造成的停顿。
GC 可扩展性：当堆内存变大时，垃圾回收器的性能如何？但大部分的程序可能并不一定关心这个问题。

1、合理化内存分配的速度、提高赋值器的 CPU 利用率

如：一次开过多的goroutine，导致赋值器的 CPU 利用率低，导致大部分在等待调度

2、降低并复用已经申请的内存 

sync.Pool 保存和复用临时对象，减少内存分配，降低 GC 压力。
sync.Pool 是可伸缩的，同时也是并发安全的，其大小仅受限于内存的大小。

sync.Pool 用于存储那些被分配了但是没有被使用，而未来可能会使用的值。这样就可以不用再次经过内存分配，可直接复用已有对象，减轻 GC 的压力，从而提升系统的性能。

常用于一些对象实例创建昂贵的场景

我们不能对 sync.Pool 中保存的元素做任何假设，以下事情是都可以发生的：

Pool 池里的元素随时可能释放掉，释放策略完全由 runtime 内部管理；
Get 获取到的元素对象可能是刚创建的，也可能是之前创建好 cache 住的。使用者无法区分；
Pool 池里面的元素个数你无法知道；
所以，只有的你的场景满足以上的假定，才能正确的使用 Pool 。sync.Pool 本质用途是增加临时对象的重用率，减少 GC 负担。

用 put 归还到 Pool 前，将对象的一些字段清零，这样，通过 Get 拿到缓存的对象时，就可以安全地使用了。

先提前分配好足够的内存，再慢慢地填充，也是一种减少内存分配、复用内存形式的一种表现。














