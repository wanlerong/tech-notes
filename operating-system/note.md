# 概述

操作系统是管理计算机硬件的程序，为应用程序提供基础，充当硬件和用户之间的中介。

计算机系统由，硬件、操作系统、系统程序和应用程序、用户组成


![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-141640.png)

计算机开启时，先运行bootstrap program（位于rom中，硬件中的固件），初始化系统，包括CPU寄存器，设备控制器，内存内容。定位操作系统内核并把它装入内存。然后开始执行第一个进程如 init，并等待事件的发生。

事件发生通过硬件或软件的中断（interrupt）来表示, 硬件通过系统总线向cpu发出信号触发interrupt, 软件通过系统调用等方式触发interrupt

操作系统是由 interrupt 驱动的。

### 存储结构：

内存（RAM random access memory）、磁盘

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-143702.png)


电子磁盘以上是易失的内存，以下是不易失的磁盘
电子磁盘本身是易失或非易失的，有内存区域，许多还有隐藏的磁盘和备份电源，外部断电时会把数据从内存写到磁盘。

### IO结构
设备控制器（如显卡）：有一定量的缓冲存储，负责在外部设备与本地缓冲存储之间进行数据传递。
设备驱动程序（如显卡驱动）：负责理解设备控制器，并提供一个设备与操作系统的统一接口。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-150321.png)

设备控制器中断驱动，适合移动少量数据。
DMA 直接内存访问，可以在设备控制器的本地缓冲和内存之间传送一整块数据。

## cpu

多核cpu，最为普遍的是对称处理，每个处理器都要完成操作系统总的所有任务。
计算机系统的设计有多种不同的方法。单处理器系统只有一个处理器，而多处理器系统包含两个或更多的处理器来共享物理存储及外设。对称多处理技术（SMP）是最为普通的多处理器设计技术，其中所有的处理器被视为对等的，且彼此独立地运行。集群系统是一种特殊的多处理器系统，它由通过局域网连接的多个计算机系统组成。


为了最好地利用CPU，现代操作系统采用允许多个作业同时位于内存中的多道程序设计，以保证CPU中总有一个作业在执行。分时系统是多道程序系统的扩展，它采用调度算法实现作业之间快速的切换，好像每个作业在同时进行一样。

只要有一个任务可以执行，cpu就不会空闲。
分时操作系统，允许多个用户同时共享cpu时间。每个用户在内存中至少有一个进程，进程执行时，通常只执行较短的时间，此时它并未完成，或者需要等待IO操作，操作系统会将cpu切换到其他用户的进程。



操作系统必须确保计算机系统的正确操作。为了防止用户干预系统的正常操作，硬件有两种模式：用户模式和内核模式。许多指令（如I／O指令和停机指令）都是特权的，只能在内核模式下执行。操作系统所驻留的内存也必须加以保护以防止用户程序修改。定时器防止无穷循环。这些工具（如双模式、特权指令、内存保护、定时器中断）是操作系统所使用的基本单元，用以实现正确操作。


进程（或作业）是操作系统工作的基本单元。进程管理包括创建和删除进程、为进程提供与其他进程通信和同步的机制。操作系统通过跟踪内存的哪部分被使用及被谁使用来管理内存。操作系统还负责动态地分配和释放内存空间，同时还管理存储空间，包括为描述文件提供文件系统和目录，以及管理大存储器设备的空间。


操作系统必须考虑到它与用户的保护和安全问题。保护是提供控制进程或用户访问计算机系统资源的机制。安全措施用来抵御计算机系统所受到的外部或内部的攻击。


分布式系统允许用户共享通过网络连接的、在地理位置上是分散的计算机的资源。可以通过客户机—服务器模式或对等模式来提供服务。在集群系统中，多个机器可以完成驻留在共享存储器上的数据的计算，即便某些集群的子集出错，计算仍可以继续。
局域网和广域网是两种基本的网络类型。局域网允许分布在较小地理区域内的处理器进行通信，而广域网允许分布在较大地理区域内的处理器进行通信。局域网通常比广域网快。


# 操作系统结构

## 系统调用
进程控制，文件管理，设备管理，信息维护，通信。

进程控制
结束，放弃 
装入，执行 
创建进程，终止进程
取得进程属性，设置进程属性
等待时间 
等待事件，唤醒事件
分配和释放内存 

文件管理 。
创建文件，删除文件 
打开，关闭 
读、写、重定位
取得文件属性，设置文件属性

设备管理
请求设备，释放设备
读、写、重定位
取得设备属性，设置设备属性
逻辑连接或断开设备

信息维护 
读取时间或日期，设置时间或日期读取系统数据，设置系统数据
读取进程，文件或设备属性。
设置进程，文件或设备属性

通信
创建，删除通信连接 。
发送，接受消息 
传递状态消息
连接或断开远程设备


# 进程管理

进程可看做是正在执行的程序。进程需要一定的资源（如CPU时间、内存、文件和I/O设备）来完成其任务。这些资源在创建进程或执行进程时被分配。进程是大多数系统中的工作单元。

进程通常还包括进程堆栈段（包括临时数据，如函数参数、返回地址和局部变量）和数据段（包括全局变量）。进程还可能包括堆（heap），是在进程运行期间动态分配的内存。

进程是活动实体，它有一个程序计数器用来表示下一个要执行的命令和相关资源集合。（还包括处理器寄存器的内容）
当一个可执行文件被装入内存时，一个程序才能成为进程。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-155856.png)

进程状态

新的：进程正在被创建。
运行：指令正在被执行。
等待：进程等待某个事件的发生（如I／O完成或收到信号）。
就绪：进程等待分配处理器。
终止：进程完成执行。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-160632.png)


## PCB
每个进程在操作系统内用进程控制块（process control block，PCB，也称为任务控制块）来表示。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-161003.png)

进程状态：状态可包括新的、就绪、运行、等待、停止等。
程序计数器：计数器表示进程要执行的下个指令的地址
CPU寄存器：根据计算机体系结构的不同，寄存器的数量和类型也不同。它们包括累加器（存储计算产生的中间结果）、索引寄存器、堆栈指针、通用寄存器和其他条件码信息寄存器。与程序计数器一起，这些状态信息在出现中断时也需要保存，以便进程以后能正确地继续执行。
CPU调度信息：这类信息包括进程优先级、调度队列的指针和其他调度参数
内存管理信息：根据操作系统所使用的内存系统，这类信息包括基址和界限寄存器的值、页表或段表（见第8章）。
记账信息：这类信息包括CPU时间、实际使用时间、时间界限、记账数据、作业或进程数量等。
IO状态信息：这类信息包括分配给进程的IO设备列表、打开的文件列表等。简而言之，PCB简单地作为这些信息的仓库，这些信息在进程与进程之间是不同的。

CPU 在进程间切换

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-161630.png)


## 进程调度

进程会被加到作业队列中，该队列包括系统中的所有进程。
驻留在内存中就绪的、等待运行的进程保存在就绪队列中。该队列通常用 PCB 的链表来实现。

linux 中的 PCB 实现：task_struct
pid_t pid                               process identifier
long state                              state of the process
unsigned int time_slice                 Scheduling information
struct files_struct \*files；        	list of open files
struct mm_struct \*mm              		address space of this process

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-170501.png)


就绪队列和各种设备队列。进程可能需要等待磁盘，因为磁盘在忙于其他进程的IO请求

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-170625.png)


![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-171326.png)

短期调度程序必须频繁地为 CPU 选择新进程。进程可能执行数毫秒（ms）就会进行IO请求，短期调度程序通常每100ms至少执行一次。由于每次执行之间的时间较短，短期调度程序必须要快。

### 上下文切换

上下文切换：CPU切换到另一个进程，需要保存当前进程的上下文，并恢复另一个进程的状态。

中断使CPU从当前任务改变为运行内核子程序，这样的操作在通用系统中发生得很频繁。当发生一个中断时，系统需要保存当前运行在CPU中进程的上下文，从而在其处理完后能恢复上下文，即先中断进程，之后再继续。进程上下文用进程的PCB表示，它包括 CPU寄存器的值、进程状态和内存管理信息等。


### 进程操作

进程在其执行过程中，能通过创建进程系统调用（create-process systemcall）创建多个新进程。
创建进程称为父进程，而新进程称为子进程。

每个新进程可以再创建其他进程从而形成了进程树。

当进程创建新进程时，有两种执行可能
- 父进程与子进程并发执行。
- 父进程等待，直到某个或全部子进程执行完

新进程的地址空间也有两种可能：
- 子进程是父进程的复制品（具有与父进程相同的程序和数据）。
- 子进程装入另一个新程序。

通过fork系统调用，可创建新进程。新进程通过复制原来进程的地址空间而成。
这种机制允许父进程与子进程方便地进行通信。两个进程（父进程和子进程）都继续执行位于系统调用fork之后的指令。但是，对于新（子）进程，系统调用fork的返回值为0；而对于父进程，返回值为子进程的id（非零）。

父进程可通过wait()来等待子进程执行完成。

通常，在系统调用fork之后，一个进程会使用系统调用exec，以用新程序来取代进程的内存空间。系统调用exec 将二进制文件装入内存，并开始执行。

当进程完成执行并使用系统调用 exit() 时，进程终止。这时，进程可以返回状态值（通常为整数）到父进程（通过系统调用wait()）。所有进程资源（包括物理和虚拟内存、打开文件和 IO 缓冲）会被操作系统释放。

父进程可以终止子进程。
- 子进程使用了超过它所分配到的一些资源。（为判定是否发生这种情况，要求父进程有一个检查其子进程状态的机制。）
- 分配给子进程的任务已不再需要。
- 父进程退出，如果父进程终止，那么有些操作系统如 VMS 不允许子进程继续

UNIX：可以通过系统调用 exit 来终止进程，父进程可以通过系统调用 wait 以等待子进程的终止。系统调用 wait 返回了终止子进程的进程标识符，以使父进程能够知道哪个子进程终止了。如果父进程终止，那么其所有子进程会以 init 进程作为父进程。因此，子进程仍然有一个父进程来收集状态和执行统计。

### 进程间通信

操作系统内并发执行的进程可以是独立进程或协作进程。如果一个进程不能影响其他进程或被其他进程所影响，那么该进程是独立的。显然，不与任何其他进程共享数据的进程是独立的。反之，该进程是协作的。显然，与其他进程共享数据的进程为协作进程。

需要进程协作的理由：
- 信息共享（information sharing）：由于多个用户可能对同样的信息感兴趣（例如共享的文件），所以必须提供环境以允许对这些信息进行并发访问。
- 提高运算速度 （computation speedup） 如果希望一个特定任务快速运行，那么必须将它分成子任务，每个子任务可以与其他子任务并行执行。注意，如果要实现这样的加速需要计算机有多个 CPU 或 IO 通道）。
- 模块化（modularity）：可能需要按模块化方式构造系统，可将系统功能分成独立进程或线程。
- 方便（convenience）：单个用户也可能同时执行许多任务。例如，一个用户可以并行进行编辑、打印和编译操作。

协作进程需要一种进程间通信机制（inter process communication，IPC）来允许进程相互交换数据与信息。
进程间通信有两种基本模式：（1）共享内存，（2）消息传递。

在共享内存模式中，建立起一块供协作进程共享的内存区域，进程通过向此共享区域读或写入数据来交换信息。
在消息传递模式中，通过在协作进程间交换消息来实现通信。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240109-233029.png)

消息传递对于交换较少数量的数据很有用，因为不需要避免冲突。对于计算机间的通信, 消息传递也比共享内存更易于实现。
共享内存允许以最快的速度进行方便的通信，在计算机中它可以达到内存的速度。
共享内存比消息传速快，消息传递通常用系统调用来实现，此需要更多的内核介入的时间消耗。
与此相反，在共享内存系统中，仅在建立共享内存区时需要系统调用，一旦建立了共享内存，所有的访问都被处理为常规的内存访问，不需要来自内核的帮助。

#### 共享内存系统

采用共享内存的进程间通信需要通信进程建立共享内存区域。通常，共享内存区域驻留在生成它的进程的地址空间。其他希望使用这个共享内存段进行通信的进程必须将此放到它们自己的地址空间上。

对于POSIX系统（一种Unix的标准）：
shmget() : 创建共享内存段，返回段的标识符
shmat() (shared memory attach): 其他进程调用，将共享内存段加入其地址空间
shmdt(): 将共享内存段从地址空间删除
shmctl(): 从系统中删除共享内存段


#### 消息传递
直接通信，进程间直接连接，通过send和receive方法传递消息
间接通信，通过邮箱或端口来发送和接收消息，进程与邮箱进行消息传递。

buffer 大小：
- 零容量：队列的最大长度为0：因此，线路中不能有任何消息处于等待。对于这种情况，必须阻塞发送，直到接收者接收到消息。
- 有限容量：队列的长度为有限的n：因此，最多只能有n个消息驻留其中。如果在发送新消息时队列未满，那么该消息可以放在队列中，且发送者可继续执行而不必等待。不过，线路容量有限。如果线路满，必须阻塞发送者直到队列中的空间可用为止
- 无限容量：队列长度可以无限，因此，不管多少消息都可在其中等待，从不阻塞发送者。


### Socket（套接字）
可定义为通信的端点。一对通过网络通信的进程需要使用一对Socket, 即每个进程各有一个。Socket由IP地址与一个端口号连接组成。服务器通过监听指定端口来等待进来的客户请求

tcp socket实现
客户端直接创建一个socket，并请求连接。
服务端 accept() 等待连接请求，返回一个socket。


### RPC (Remote procedure call)
RPC是另一种形式的分布式通信。当一个进程（或线程）调用一个远程应用的方法时，就出现了RPC。


# 线程

线程是CPU使用的基本单元，它由线程ID、程序计数器、寄存器集合和栈组成。它与属于同一进程的其他线程共享代码段、数据段和其他操作系统资源，如打开文件和信号。也会共享内存

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240110-143831.png)

栈：是个线程独有的，保存其运行状态和局部自动变量的。栈在线程开始的时候初始化，每个线程的栈互相独立，因此，栈是 thread safe的。操作系统在切换线程的时候会自动的切换栈，就是切换 ＳＳ／ＥＳＰ寄存器。

堆：与进程相关，在操作系统对进程初始化的时候分配，每一个线程访问共同的堆。

动机：一个进程可能需要执行多个相似的任务，多线程可以减少开销，并行处理。
模块化，不同线程处理不同的任务。


优点：
- 响应度高：即使其部分阻塞或耗时较长，也不影响整体响应。能通过另一个线程与用户交互。
- 资源共享：线程默认共享它们所属进程的内存和资源。代码和数据共享的优点是它能允许一个应用程序在同一地址空间有多个不同的活动线程。
- 经济：进程创建所需要的内存和资源的分配比较昂贵。由于线程能共享它们所属进程的资源，所以创建和切换线程会更为经济。
- 可以利用多核CPU，可以同时运行在多个CPU上。

线程库：
pthread等线程库，一般都有创建线程方法，join方法（等待线程执行完成）。



多线程问题：
一个线程调用了fork，会创建新进程。然后有两种形式，复制所有线程 或者 只复制调用fork的线程。
如果一个线程调用了exec，会替换整个进程包括所有线程。


linux中通常是task，而不是进程或线程。
linux也提供了clone来创建线程，clone可以通过参数设置共享的范围（如内存空间，打开的文件，信号处理程序等）。
当clone不共享时，和fork的作用就一样了。
类似于对task_struct数据的处理。fork是深拷贝，clone方法中共享的部分是浅拷贝。


线程取消

线程取消（thread cancellation）是在线程完成之前来终止线程的任务。

要取消的线程通常称为目标线程。目标线程的取消可在如下两种情况下发生
1）异步取消（asynchronous cancellation）：一个线程立即终止目标线程。操作系统回收取消线程的系统资源，但是通常并不回收所有资源。
2 延取消（deferred cancellation）：目标线程不断地检查它是否应终止，这允许目标线程有机会以有序方式来终止自己。这允许一个线程检查它是否是在安全的点被取消。


### 信号

在UNIX中用来通知 进程 某个特定事件已发生了。根据需要通知信号的来源和事件的理由，信号可以同步或异步接收。不管信号是同步或异步的，所有信号具有同样模式
- 信号是由特定事件的发生所产生的。
- 产生的信号要发送到进程。
- 一旦发送，信号必须加以处理。

同步信号（内部产生的）的例子包括非法访问内存或被0所除。如果运行程序执行这些动作，那么就产生信号。同步信号发送到执行操作而产生信号的同一进程。

当一个信号由运行进程之外的事件产生，那么进程就异步接收这一信号。这种信号的例子包括使用特殊键（按如Crl+C键）或定时器到期。通常，异步信号被发送到另一个进程。


- 默认信号处理程序
- 用户定义的信号处理程序。

每个信号都有一个默认信号处理程序（default signal handler），当处理信号时是在内
核中运行的。这种默认动作可以用用户定义的信号处理程序来改写。信号可按不同的方式
处理。有的信号可以简单地忽略（如改变窗口大小），其他的（如非法内存访问）可能要通过终止程序来处理。

单线程程序的信号处理比较直接，信号总是发送给进程。不过，对于多线程程序，发
送信号通常有如下选择：
- 发送信号到信号所应用的线程。
- 发送信号到进程内的每个线程。
- 发送信号到进程内的某些固定线程
- 规定一个特定线程以接收进程的所有信号。


同步信号需要发送到产生这一信号的线程，而不是进程的其他线程。

有的异步信号如终止进程的信号（例如按Ctrl+C键）应该发送到所有线程。
大多数多线程版UNIX允许线程描述它会接收什么信号和拒绝什么信号。因此，有时一个异步信号只能发送给那些不拒绝它的线程。

不过，因为信号只能处理一次，所以信号通常发送到不拒绝它的第一个线程。标准的发送信号的UNIX函数是kill(pid_t pid ,int signal)，在这里指定了信号的发送进程 pid。不过POSIX Pthread还提供了 pthread_kill(pthread_t tid, int signal）函数，此函数允许信号被传送到一个指定的线程（tid）。

### 线程池
如果允许所有并发请求都通过新线程来处理，那么将没法限制在系统中并发执行的线程的数量。无限制的线程会耗尽系统资源，如CPU时间和内存。解决这个问题的一种方法是使用线程池（threadpool）。

线程池的主要思想是在进程开始时创建一定数量的线程，并放入到池中以等待工作。当服务器收到请求时，它会唤醒池中的一个线程（如果有可以用的线程），并将要处理的请求传递给它。一旦线程完成了服务，它会返回到池中再等待工作。如果池中没有可用的线程，那么服务器会一直等待直到有空线程为止。

优点
1）通常用现有线程处理请求要比等待创建新的线程要快
2) 线程池限制了在任何时候可用线程的数量。这对那些不能支持大量并发线程的系统非常重要。

线程池中的线程数量由系统CPU的数量、物理内存的大小和并发客户请求的期望值等因素决定。比较高级的线程池能动态调整线程的数量，以适应具体情况。这类结构的优点是在系统负荷低时减低内存消耗。

## 线程模型

多对一：如果一个线程执行了阻塞的系统调用，那么整个进程会阻塞。多核cpu不能发挥作用
![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240110-155949.png)


一对一：可以并发执行系统调用，但是每创建一个用户线程就需要创建一个内核线程。会限制系统所支持的线程数量。
![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240110-160001.png)


多对多：可以并发执行，阻塞时，内核会调度另一个内核线程来执行。
![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240110-160020.png)


用户线程对程序员来说是可见的，而对内核来说却是未知的。操作系统支持和管理内核线程。通常，用户线程跟内核线程相比，创建和管理要更快，因为它不需要内核干预。


# CPU 调度

就绪队列可实现为FIFO队列、优先队列、树或简单的无序链表。队列中的记录通常为进程控制块（PCB）。

### 抢占调度

- 当一个进程从运行状态切换到等待状态（例如，IO请求，或调用wait等待一个子进程的终止)
- 当一个进程从运行状态切换到就绪状态（例如，当出现中断时）。
- 当一个进程从等待状态切换到就绪状态（例如，IO完成）。
- 当一个进程终止时

对于第1和第4两种情况，没有选择而只有调度。
采用非抢占调度，一且CPU分配给一个进程，那么该进程会一直使用CPU直到进程终止或切换到等待状态。

### 分派程序 dispatcher
负责将cpu的控制，交给被调度程序选择的进程。功能：
- 上下文切换
- 切换到用户模式
- 跳转到用户程序的合适位置，以重新启动程序

### 调度准则
用于比较调度算法。

- cpu使用率。需要使 cpu 尽可能忙
- 吞吐量。一个时间单位内所完成进程的数量
- 周转时间：从进程提交到进程完成的时间段。周转时间为所有时间段之和，包括等待进入内存、在就绪队列中等待、在CPU上执行和IO执行。
- 等待时间：CPU调度算法并不影响进程运行和执行IO的时间：它只影响进程在就绪队列中等待所花的时间。
- 响应时间：对于交互系统，是从提交请求到产生第一响应的时间。这种时间称为响应时间，是开始响应所需要的时间，而不是输出响应需要的时间

## 调度算法

### 先到先服务调度 first-come first-served  FCFS
先请求 cpu 的进程先分配到 cpu。用链表就能实现。
问题在于，平均等待时间会比较长，如果耗时长的进程先被调度的话。
假设有一个很耗CPU，只吃一点IO的进程，和很多和很耗IO但只吃一点CPU的进程，那每次调度到耗CPU进程时，其余所有的都得等待，IO设备将会空闲。

先到先服务调度算法是非抢占的。一且CPU被分配给了一个进程，该进程就会保持CPU直到释放CPU为止，即程序终止或是请求IO。FCFS算法对于分时系统（每个用户需要定时地得到一定的CPU时间）是特别麻烦的。允许一个进程保持 CPU 时间过长将是个严重错误。

### 最短作业优先调度（SJF）

最佳的，因为平均等待时间最小。

但问题是无法知道cpu区间的长度。只可以近似的预测，如下一个长度和之前的相似。

预测值 = a * 最近的一次观测到的值 + (1-a) * 上一个预测值 (表示过去的历史)

可能是抢占的或非抢占的。当一个新进程到达就绪队列而以前进程正在执行时，就需要选择。与当前运行的进程相比，新进程可能有一个更短的CPU区间。抢占SJF算法可抢占当前运行的进程，而非抢占SJF算法会允许当前运行的进程先完成其CPU区间。

抢占SJF调度有时称为最短剩余时间优先调度（shortest-remaining-time-first scheduling ）

### 优先级调度
SJF就是优先级调度的一个特例。

优先级可通过内部或外部方式来定义。内部定义优先级使用一些测量数据以计算进程优先级。例如，时间极限、内存要求、打开文件的数量和平均IO区间与平均CPU区间之比都可以用于计算优先级。

外部优先级是通过操作系统之外的准则来定义的，如进程重要性等等。

问题
- 无穷等待（饥饿）：会使得某个低优先级的进程，无穷等待CPU，一直无法执行

解决方法：老化，即逐渐增加在系统中等待很长时间的进程的优先级

### 轮转法调度 round-robin RR

为分时系统设计的（每个用户需要定时地得到一定的CPU时间），类似于先到先服务调度，但是增加了抢占以切换进程。

定义一个较小时间单元，称之为时间片, 时间片通常为10 - 100 ms，将就绪队列作为循环队列。 CPU 调度程序循环就绪队列，为每一个进程分配不超过一个时间片的CPU。


为了实现RR调度，将就绪队列保存为进程的FIFO队列，新进程增加到就绪队列的尾部，CPU调度程序从就绪队列中选择第一个程序，设置定时器在一个时间片之后中断，再分派该进程。
接下来将可能发生两种情况，进程可能只需要小于时间片的CPU区间，对于这种情况，进程本身会自动释放CPU，调度程序接着处理就绪队列的下一个进程，否则，如果当前运行进程的CPU区间比时间片要长，定时器会中断并产生操作系统中断，然后进程上下文切换，将进程加入到就绪队列的尾部，接着CPU调度程序会选择就绪队列中的下一个进程。

RR调度算法是可抢占的，对于RR调度算法，队列中没有进程被分超过一个时间片的CPU时间(除非它是唯一可运行的进程)，如果进程的CPU区间超过了一个时间片，那么该进程会被抢占，而被放回就绪队列。

RR算法的性能很大程度上依赖域时间片的大小，在极端情况下，如果时间片非常大，那么RR算法与FCFS算法一样，如果时间片很小(如毫秒)，那么RR算法称为处理器共享，（从理论上来说）n个进程对于用户都有它自己的处理器，速度为真正处理数度的 1/n。

时间片越小，上下文切换的次数就会越多。浪费的时间比例 = 上下文切换耗时 / 时间片的长度

绝大多数现代操作系统的低间分配为10～100ms，上下文切换的时间一般少于10us。因此，上下文切换的时间仅占时间片的一小部分。

### 多级队列调度

多级队列调度算法将就绪队列分成多个独立队列，根据进程的属性，如内存大小，进程优先级、进程类型、一个进程被永久地分配到一个队列，每个队列有自己的调度算法。这种算法的优点是低调度开销，缺点是不够灵活。

比如:如下几个队列，优先级从高到低
- 系统进程
- 交互进程
- 交互编辑进程
- 批处理进程
- 学生进程


### 多级反馈队列调度
多级反馈队列调度算法，允许进程在队列之间移动，主要思想是根据不同CPU区间的特点以区分进程，如果进程使用过多的CPU时间，那么它会被移到更低的优先级队列。
![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240111-223715.png)




## 多处理器调度的方法

在一个多处理器中，CPU调度的一种方法是让一个处理器(主服务器)处理所有的调度决定、I/O处理以及其他系统活动，其他的处理器只执行用户代码。这种非对称多处理方法更为简单，因为只有一个处理器访问系统数据结构，减轻了数据共享的需要。

另一种方法是使用对称处理器方法（SMP），即每个处理器自我调度，所有进程可能处于一个共同的就绪队列中，或每个处理器都有它自己的私有就绪队列。无论如何，调度通过每个处理器检查共同就绪队列并选择一个进程来执行。如果多处理器试图访问和更新一个共同数据结构，那么每个处理器必须编程：必须保证两个处理器不能选择同一个进程，且进程不会从队列中丢失。

处理器亲和性
考虑一下，当一个进程在一个特定处理器上运行时，缓存中会发生什么？进程最近访问的数据进入处理器缓存，结果是进程所进行的连续内存访问通常在缓存中得以满足。现在可以考虑一下，如果进程移到其他处理器上时，会发生什么：被迁移的第一个处理器的缓存中的内容必须为无效，而将要迁移的第二个处理器的缓存需要重新构建。由于使缓存无效或重新构建的代价高，绝大多数SMP系统试图避免将进程从一个处理器移至另一个处理器，而是努力使一个进程在同一个处理器上运行，这被称为处理器亲和性，即一个进程需要对其运行所在的处理器的亲和性。

负载平衡
负载平衡 设法将工作负载平均地分配到SMP系统中的所有处理器上，值得注意的是，负载平衡通常只是对那些拥有自己私有的可执行进程的处理器而言是必要的。在具有共同队列的系统中，通常不需要负载平衡，因为一但处理器空闲，它立刻从共同队列中取走一个可执行进程，但同样值得注意的是，在绝大多数支持SMP中，每个处理器都具有一个可执行进程的私有队列。
负载平衡常会抵消处理器的亲和性的优点。

## linux上的CPU调度

给高优先级分配较长的时间片，低优先级分配较短的时间片
![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240111-224645.png)

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240111-225052.png)

活动队列：时间片还未耗尽的队列。
到期队列：当任务耗尽其时间片后，它被认为是到期了

由于对SMP的支持，每个处理器维护它自己的运行队列，并独立地调度它自己。每个运行队列包括两个优先级队列——活动的和到期的。每个优先级队列都有一个根据其优先级索引的任务列表。调度程序从活动队列中选择最高优先级的任务来在CPU上执行。对于多处理器系统，这意味着每个处理器从其自己的运行队列中调度最高优先级的任务。当所有任务都耗尽其时间片（即活动队列为空）时，两个优先级队列相互交换，到期队列变为活动队列，反之亦然。

实时任务被分配静态优先级，所有其他任务都具有动态优先级，并基于它们自己的nice值加上或减去5。任务的交互性决定了从nice值中加上还是减去5，而任务的交互性决定于它在等待IO时沉睡了多长时间。交互性更强的任务通常具有更长的沉睡时间，因此更可能按-5来调整，因为调度程序偏爱交互式任务。

睡眠时间短的任务通常更受CPU制约，优先级会更低。

# 内存管理


# 存储管理

## IO

# 保护与安全

# 分布式系统

# 特殊用途系统
