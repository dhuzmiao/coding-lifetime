> 众所周知,redis是目前非常流行的缓存中间件之一。在redis官网有这么一段话: redis有着丰富的数据结构，如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets） 与范围查询， bitmaps， hyperloglogs 和 地理空间（geospatial） 索引半径查询。Redis 内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的 磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（high availability）。本文主要整理自redis官方文档。

这篇文章我们主要来整理下redis的复制（replication）和两种高可用方案 Redis 哨兵（Sentinel）和自动分区（Cluster）。

# 复制（replicaiton）
## 说明
在 Redis 复制的基础上，主从复制非常容易使用和复制，其保证 slave Redis server 能精确地复制 master redis server 上的内容。每次当 slave 和 master 之间连接断开时，slave 会自动重连到 master 上，并且无论这期间 master 发生了什么，slave 都将始终确保让自己成为 master 的精确副本。
这个系统运行依靠三个主要的机制：
1. 当一个 master 实例和一个 slave 实例连接正常时，master 会发送一连串的命令来保持对 slave 的更新，以便于将自身数据集的改变复制给 slave（包括客户端的写入、key 的过期或被逐出等等）。
2. 当 master 和 slave 之间的连接断开之后，因为网络的问题、或者是主从意识到连接超时，slave 重新连接上 master 并会尝试进行部分重分布（这意味着它会尝试只获取在断开连接期间内丢失的命令流）。
3. 当无法进行部分重同步时，slave 会请求的进行全量重同步。这会涉及到一个更复杂的过程，例如 master 需要创建所有数据的快照，将之发送给 slave，之后在数据集更改时持续发送命令流到 slave 。

Redis 使用默认的异步复制，其特点是低延迟和高性能，是绝大多数 Redis 用例的自然复制模式。但是，从 Redis 服务器会异步地确认其从主 Redis 服务器周期接收到的数据量。

客户端可以使用 WAIT 命令来请求同步复制某些特定的数据。但是，WAIT 命令只能确保在其它 Redis 实例中有指定数量的已确认的副本（在故障转移期间，由于不同原因的故障转移或是由于 Redis 持久性的实际配置，故障转移期间确认的写入操作可能仍然会丢失）。你可以查看 Sentinel 或者 Reids 集群文档，了解关于高可用性和故障转移的更多信息。

接下来的是一些关于 Redis 复制非常重要的事实：
- Redis 使用异步复制， slave 和 master 之间异步地确认处理的数据量
- 一个 master 可以拥有多个 slave
- slave 可以接受其它 slave 的连接。除了多个 slave 可以连接到同一个 master 之外，slave 之间也可以像层叠状的结构（cascading-like structure）连接到其它 slave。自 Redis 4.0 起，所有的 sub-slave 将会从 master 接收到完全一样的复制流。
- Redis 复制在 master 侧是非阻塞的。这意味着 master 在一个或多个 slave 进行初次同步或者是部分重同步时，可以继续处理查询请求。
- 复制在 slave 侧大部分也是非阻塞的。当 slave 进行初次同步时，你可以在 redis.conf 里配置使用旧数据集处理查询请求。否则，你可以配置如果复制流断开，Redis slave 会返回一个 error 给客户端。但是，在初次同步之后，旧数据集必须被删除，同时加载新的数据集。slave 在这个短暂的时间窗口内（如果数据集很大，会持续较长时间），会阻塞到来的链接请求。自 Redis 4.0 开始，可以配置 Redis 使用删除旧数据集的操作在另一个不同的线程中进行，但是，加载新数据集的操作依然需要在主线程中进行并且会阻塞 slave 。
- 复制既可以被用在弹性伸缩时，以便只读查询可以有多个 slave 进行（例如O(N)复杂度的慢操作可以被下放到 slave ），或者仅用于数据安全。
- 可以使用复制来避免 master 将全部数据集写入磁盘造成的开销（一种典型的技术是配置你的 master Redis.conf 以便避免对磁盘进行持久化，然后连接一个 slave ， 其配置为不定期保存或是启用 AOF ）。但是，这个设置必须小心处理，因为重新启动的 master 程序将从一个空数据集开始（如果一个 slave 试图与它同步，那么这个 slave 也会被清空）。

## 当 master 关闭持久化时，复制的安全性
在使用 Redis 复制功能的设置中，强烈建议在 master 和 slave 中启用持久化。当不可能启用时，例如由于非常慢的磁盘性能而导致的延迟问题，**应该配置实例来避免重置后自动重启**。
为了更好地理解为什么关闭了持久化并配置了自动重启的 master 是危险的，检查以下故障模式，这些故障模式中数据会从 master 和 所有 slave 中被删除：
1. 我们设置节点 A 为 master 并关闭它的持久化设置，节点 B 和 C 从节点 A 复制数据。
2. 节点 A 崩溃，但是它有一些自动重启的系统可以重启进程。但是由于持久化被关闭了，节点重启后其数据集为空。
3. 节点 B 和节点 C 会从节点 A 复制数据，但是节点 A 的数据集是空的，因此复制的结果是它们会销毁自身之前的数据副本。

当 Reids Sentinel 被用于高可用并且 master 关闭持久化，这时如果允许自动重启的进程也是很危险的。例如，master 可以重启的足够快以至于 Sentinel 没有探测到故障，因此上述的故障模式也会发生。
任何时候数据安全性都是最重要的，所有如果 master 使用复制功能同时没有做持久化配置，**那么自动重启进程这项应该被禁用**。

## Reids 复制功能是如何工作的
每一个 Redis master 都有一个 replication ID（这是一个较大的伪随机字符串，标记了一个给定的数据集）。每个 master 也持有一个偏移量，master 将自己产生的复制流发送给 slave 时，发送多少个字节的数据，自身的偏移量就会增加多少，目的是当有新的操作修改自己的数据集时，它可以以此更新 slave 的状态。复制偏移量即使在没有一个 slave 连接到 master 时，也会自增，所以基本上每一对给定的 **Replication ID, offset** 都会表示一个 master 数据集的确切版本。

> TBD

# Redis 哨兵（Sentinel）
## 说明
Redis 的 Sentinel 系统用于管理多个 Redis instance ，该系统执行以下三个任务：
- **监控（Monitoring）**： Sentinel 会不断地检查你的主服务器和从服务器是否运行正常。
- **提醒（Notification）**： 当被监控的某个 Redis 服务器出现问题时，Sentinel 可以通过 API 向管理员或者其它应用程序发送通知。
- **自动故障迁移（Automatic failover）**： 当一个主服务器不能正常工作时，Sentinel 会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其它从服务器改为复制新的主服务器；当客户端试图连接失效的主服务器时，集群也会向客户端返回新主服务器的地址，使得集群可以使用新主服务器代替失效服务器。

Redis Sentinel 是一个分布式系统，你可以在一个架构中运行多个 Sentinel 进程（progress），这些进程使用 gossip protocols 来接收关于主服务器是否下线的信息，并使用投票协议（agreement protocols）来决定是否执行自动故障迁移，以及选择哪个从服务器作为新的主服务器。

虽然 Redis Sentinel 释放出一个单独的可执行文件 redis-sentinel，但实际上它只是一个运行在特殊模式下的 Redis 服务器，你可以在启动一个普通 Redis 服务器时通过给定 `-sentinel` 选项来启动 Redis Sentinel 。

> TBD

# 自动分区（Cluster）
## 说明
本文档是Redis集群的一般介绍，没有涉及复杂难懂的分布式概念的赘述，只是提供了从用户角度来如何搭建测试以及使用的方法，如果你打算使用并深入了解Redis集群，推荐阅读完本章节后,仔细阅读 Redis 集群规范 一章。

## Redis 集群介绍
Reids 集群是一个提供在**多个 Redis 间节点间共享数据**的程序集。
Redis 集群并不支持处理多个 keys 的命令，因为这需要在不同的节点间移动数据，从而达不到像 Redis 那样的性能，在高负载的情况下可能会导致不可预料的错误。
Reids 集群通过分区提供**一定程度的可用性**，在实际环境中当某个节点宕机或者不可达的情况下继续处理命令。
Redis 集群的优势：
- 自动分割数据到不同的节点上。
- 整个集群的部分节点失败或者不可达的情况下能够继续处理命令。

## Redis 集群的数据分片
Redis 集群没有使用一致性哈希，而是引入了**哈希槽**的概念。
Redis 集群有16384个哈希槽，每个 key 通过 CRC16 校验后对16384取模来决定放置哪个槽。集群的每个节点负责一部分哈希槽，举个例子，比如当前集群有3个节点，那么：
- 节点 A 包含 0 到 5500 号哈希槽
- 节点 B 包含 5501 到 11000 号哈希槽
- 节点 C 包含 11001 到 16384 号哈希槽
这个节点结构很容易添加或者删除节点，比如如果我想添加个新节点 D，我需要从节点 A，B，C 中分部分槽到 D 上。如果我想移除节点 A，需要将 A 中的槽移动到 B 和 C 节点上，然后将没有任何槽点的 A 节点从集群中移除即可。由于从一个节点将哈希槽移动到另一个节点并不会停止服务，所以无论添加删除或者改变某个节点的哈希槽的数据都不会造成集群不可用的状态。

> TBD



# 参考文档
- 浅析redis主从、哨兵和Cluster：https://cloud.tencent.com/developer/article/1499541
- http://www.redis.cn/topics/replication.html
- https://blog.csdn.net/c295477887/article/details/52487621
- http://www.redis.cn/topics/sentinel.html
