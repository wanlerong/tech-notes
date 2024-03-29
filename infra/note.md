# 监控指标：

## 如何监控系统中的这些指标，并且排查问题，debug。？

Response Time:
Monitor the average response time for API endpoints or services. 

Error Rates: SLA
Track the rate of errors or failures in your backend service. 

QPS 请求吞吐量：
测量每单位时间处理的请求数。 了解流量模式有助于容量规划和识别潜在峰值。
不同服务之间的负载。

资源利用率：(后端服务的机器，数据库，es，redis，kafka 等)
监控 CPU、内存和磁盘使用情况，确保资源得到充分配置。 在潜在的资源瓶颈影响性能之前识别它们。
监控资源使用趋势以规划未来的容量需求。

数据库性能：
iops 每秒的IO，跟踪数据库响应时间、查询执行时间和连接池利用率
慢查询

数据库连接数：
记得是 2000

iops是：8000


网络延迟：
测量不同组件或服务之间的网络延迟。 高延迟会影响整体系统性能。


GC 指标：
如果使用具有自动内存管理功能的语言，请监视垃圾收集指标以确保有效的内存使用并最大程度地减少中断。
队列：
如果使用消息队列或异步处理，请监视队列是否有消息积压。 

依赖健康：
监控外部依赖项的运行状况，例如第三方 API 或服务。

响应代码分布：
分析HTTP响应码的分布。 识别可能表明客户端或服务器端问题的模式，例如 4xx 或 5xx 代码的增加。

业务指标：
监控与关键业务流程一致的指标。 例如，跟踪用户注册、交易或其他关键业务事件。

Chaos engineering 压测

# 分布式

CAP 原理：分布式系统无法同时确保一致性（Consistency）、可用性（Availability）和分区容忍性（Partition），设计中往往需要弱化对某个特性的需求。

一致性、可用性和分区容忍性的具体含义如下：
一致性（Consistency）：等同于所有节点访问同一份最新的数据副本。任何事务应该都是原子的，所有副本上的状态都是事务成功提交后的结果，并保持强一致；
可用性（Availability）：每次请求都能获取到非错的响应——但是不保证获取的数据为最新数据。。系统（非失败节点）能在有限时间内完成对操作请求的应答；
分区容忍性（Partition）：系统中的网络可能发生分区故障（成为多个子网，甚至出现节点上线和下线），即节点之间的通信无法保障。而网络故障不应该影响到系统正常服务。

弱化一致性
对结果一致性不敏感的应用，可以允许在新版本上线后过一段时间才最终更新成功，期间不保证一致性

弱化可用性
对结果一致性很敏感的应用，例如银行取款机，当系统故障时候会拒绝服务。

弱化分区容忍性
现实中，网络分区出现概率较小，但很难完全避免。
两阶段的提交算法，某些关系型数据库以及 ZooKeeper 主要考虑了这种设计。


# 分布式事务

https://icyfenix.cn/architect-perspective/general-architecture/transaction/distributed.html

## 二阶段提交：
因此，二阶段提交的算法思路可以概括为：参与者将操作成败通知协调者，再由协调者根据所有参与者的反馈情况决定各参与者是要提交操作还是中止操作。

两个阶段是指：
第一阶段：准备阶段(投票阶段voting phase)
第二阶段：提交阶段(执行阶段commit phase)

该协议由两个阶段组成：

提交请求阶段（或投票阶段），其中协调程序进程尝试准备所有事务的参与进程（指定的参与者、队列或工作人员），以采取必要的步骤来提交或中止事务并进行投票“是”：提交（如果事务参与者的本地部分执行已正确结束），或“否”：中止（如果检测到本地部分存在问题），

提交阶段，协调者根据参与者的投票决定是否提交（仅当所有人都投“是”时）或中止事务（否则），并将结果通知所有参与者。然后，参与者使用其本地事务资源（也称为可恢复资源；例如数据库数据）以及事务的其他输出（如果适用）中的相应部分来执行所需的操作（提交或中止）。


Commit request (or voting) phase
The coordinator sends a query to commit message to all participants and waits until it has received a reply from all participants.
The participants execute the transaction up to the point where they will be asked to commit. They each write an entry to their undo log and an entry to their redo log.
Each participant replies with an agreement message (participant votes Yes to commit), if the participant's actions succeeded, or an abort message (participant votes No to commit), if the participant experiences a failure that will make it impossible to commit.
Commit (or completion) phase

Success
If the coordinator received an agreement message from all participants during the commit-request phase:

The coordinator sends a commit message to all the participants.
Each participant completes the operation, and releases all the locks and resources held during the transaction.
Each participant sends an acknowledgement to the coordinator.
The coordinator completes the transaction when all acknowledgements have been received.


Failure
If any participant votes No during the commit-request phase (or the coordinator's timeout expires):

The coordinator sends a rollback message to all the participants.
Each participant undoes the transaction using the undo log, and releases the resources and locks held during the transaction.
Each participant sends an acknowledgement to the coordinator.
The coordinator undoes the transaction when all acknowledgements have been received.


缺点：

两阶段提交协议的最大缺点是它是一个阻塞协议。
如果协调器永久失败，某些参与者将永远无法解决其事务：在参与者发送协议消息作为对协调器提交请求消息的响应之后，它将阻塞，直到收到提交或回滚。
两阶段提交协议无法在提交阶段从协调器和队列成员的故障中可靠地恢复。


## TCC 事务 Try-Confirm-Cancel

- Try：尝试执行阶段，完成所有业务可执行性的检查（保障一致性），并且预留好全部需用到的业务资源（保障隔离性）。
- Confirm：确认执行阶段，不进行任何业务检查，直接使用 Try 阶段准备的资源来完成业务处理。Confirm 阶段可能会重复执行，因此本阶段所执行的操作需要具备幂等性。
- Cancel：取消执行阶段，释放 Try 阶段预留的业务资源。Cancel 阶段可能会重复执行，也需要满足幂等性。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/screenshot-20240315-113628.png)


由上述操作过程可见，TCC 其实有点类似 2PC 的准备阶段和提交阶段，但 TCC 是位于用户代码层面，而不是在基础设施层面，这为它的实现带来了较高的灵活性，可以根据需要设计资源锁定的粒度。

TCC 在业务执行时只操作预留资源，几乎不会涉及锁和资源的争用，具有很高的性能潜力。
但是 TCC 并非纯粹只有好处，它也带来了更高的开发成本和业务侵入性，意味着有更高的开发成本和更换事务实现方案的替换成本，所以，通常我们并不会完全靠裸编码来实现 TCC，而是基于某些分布式事务中间件（譬如阿里开源的Seata）

优点：XA两阶段提交资源层面的，而TCC实际上把资源层面二阶段提交上提到了业务层面来实现。有效了的避免了XA两阶段提交占用资源锁时间过长导致的性能地下问题。
缺点：主业务服务和从业务服务都需要进行改造，从业务方改造成本更高。而TCC中需要提供Try-Confirm-Cancel三个接口，大大增加了开发量。

# 高并发

## 水平缩放：
不是通过向单台计算机添加更多资源来垂直扩展，而是通过向基础设施添加更多计算机来水平扩展。 这种方法提高了容错能力，并允许您处理更多并发请求。

## 容器化和编排：
使用 Docker 等技术对您的应用程序进行容器化，并使用 Kubernetes 等工具对其进行编排。 这简化了服务的部署、扩展和管理。

## 缓存：
对经常访问的数据实施缓存机制，以减少后端服务的负载。 使用分布式缓存解决方案来提高可扩展性。

## 内容分发网络（CDN）：
利用 CDN 缓存静态资产并将其交付到更靠近最终用户的位置，从而减少延迟并减轻服务器的流量。

## 异步处理：
将耗时的任务卸载到后台作业或队列。 这允许您的主应用程序快速响应传入的请求，同时单独处理后台任务。

## 数据库优化：
优化数据库查询，使用索引，并考虑使用数据库复制或分片策略以获得更好的读写性能。 利用缓存来存储数据库查询结果。


## 监控和记录：
实施全面的监控和记录，以主动识别和解决问题。 使用 Prometheus、Grafana、ELK stack 等工具来深入了解系统的性能。

自动缩放：
根据流量模式实施自动缩放。 根据需求自动添加或删除资源，以确保最佳性能和成本效率。

混沌工程：
定期进行混沌工程实验来模拟系统中的故障和弱点。 这有助于在潜在问题影响用户之前识别它们。

安防措施：
实施强大的安全措施来防范常见威胁，例如 DDoS 攻击，并定期审核您的系统是否存在漏洞。

全球分布：
如果您的应用程序拥有全球用户群，请考虑跨多个地理区域分布您的服务，以减少延迟并提高容错能力。

持续测试和部署：
采用持续集成和持续部署（CI/CD）方法快速安全地发布新功能和错误修复。

## 限流
实施节流和速率限制机制，以防止滥用并确保资源的公平使用。 这可以帮助保护您的后端免受流量突然激增的影响。

## 服务降级：
如果用户或客户端超出限制，请提供降级但仍然可用的体验，而不是完全阻止访问。

## 性能分析工具
考虑使用性能分析工具来识别和解决瓶颈。内存管理：有效管理内存，防止内存泄漏并确保最佳的资源利用率。 考虑使用内存分析器等工具来识别和解决与内存相关的问题。

并发性和并行性：利用并发性和并行性同时处理多个请求

连接池：为数据库连接实现连接池。重用现有连接而不是为每个请求创建新连接可以显着提高性能；对外部服务使用连接池，例如与其他 API 的 HTTP 连接。 这有助于管理和重用连接，减少为每个请求建立新连接的开销。

压缩：在通过网络发送数据之前对其进行压缩，以减少带宽使用并缩短响应时间。 使用 Gzip 或 Brotli 等压缩算法。

分布式追踪：Distributed Tracing:实施分布式跟踪来监视和跟踪请求在不同的微服务中移动时的情况。 这有助于识别性能瓶颈并解决问题。

监控和记录：使用强大的监控和日志记录工具来深入了解后端服务的性能和运行状况。 在问题影响用户之前主动解决问题。

# 高可用

## 微服务架构：将您的应用程序分解为更小的、独立的微服务。 这使您可以独立扩展各个组件，并确保一项服务的故障不一定会影响整个系统。


## 故障转移和冗余：
设计具有冗余和故障转移机制的系统。 使用多个数据中心或区域，确保部分服务崩溃或物理机损坏不会影响整体可用性。