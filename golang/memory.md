# 内存管理 - data race

https://go.dev/ref/mem

go memory model 指明了在什么条件下，一个 goroutine 读取一个 variable 时，能保证可以读取到，另一个 goroutine 对同一个 variable 的写入。


修改由多个 goroutine 同时访问的数据时，必须串行。通过 channel，或者 sync、atomic。

A data race 是一个 goroutine 对内存位置的 write，和另一个 goroutine 对同一内存位置的 read 或 write 同时发生。


一个 memory operation 包含四个细节：
- 类型，是普通读，普通写，还是同步的操作比如，atomic access，mutex operation, channel operation
- 在程序中的位置
- 内存的位置
- 对应的读写的值

read-like memory operation: 包含 read，atomic read，mutex lock，channel receive
write-like memory operation: write, atomic write, mutex unlock, channel send, channel close.
both read-like and write-like: atomic compare-and-swap

单个 goroutine 的执行，可以被建模为一组内存操作的集合

要求：
在每一个 goroutine 中，内存操作必须按对应的顺序执行
一个 go 程序的执行，就是一组 goroutine 的执行，还包含一个 mapping W，是 W(r) = w，read-like operation 映射到对应的 the write-like operation



The go statement that starts a new goroutine is synchronized before the start of the goroutine's execution.
The exit of a goroutine is not guaranteed to be synchronized before any event in the program.

channel

A send on a channel is synchronized before the completion of the corresponding receive from that channel.
The closing of a channel is synchronized before a receive that returns a zero value because the channel is closed. 
A receive from an unbuffered channel is synchronized before the completion of the corresponding send on that channel.
The kth receive on a channel with capacity C is synchronized before the completion of the k+Cth send from that channel completes.

lock

For any sync.Mutex or sync.RWMutex variable l and n < m, call n of l.Unlock() is synchronized before call m of l.Lock() returns.


sync.once

The completion of a single call of f() from once.Do(f) is synchronized before the return of any call of once.Do(f).


Atomic Values

The APIs in the sync/atomic package are collectively “atomic operations” that can be used to synchronize the execution of different goroutines. If the effect of an atomic operation A is observed by atomic operation B, then A is synchronized before B. 


Finalizers

The runtime package provides a SetFinalizer function that adds a finalizer to be called when a particular object is no longer reachable by the program. A call to SetFinalizer(x, f) is synchronized before the finalization call f(x).



Programs with races are incorrect and can exhibit non-sequentially consistent executions. 


Not introducing data races also means not assuming that loops terminate. 
Not introducing data races also means not assuming that called functions always return or are free of synchronization operations. 

