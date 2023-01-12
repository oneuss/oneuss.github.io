---
layout: post
title:  "Zookeeper笔记"
subtitle: ""
date:   2020-04-22
background: '/img/imac_bg.png'
---
# Zookeeper是什么？
看[官方描述](https://zookeeper.apache.org/)：
>ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. All of these kinds of services are used in some form or another by distributed applications. Each time they are implemented there is a lot of work that goes into fixing the bugs and race conditions that are inevitable. Because of the difficulty of implementing these kinds of services, applications initially usually skimp on them, which make them brittle in the presence of change and difficult to manage. Even when done correctly, different implementations of these services lead to management complexity when the applications are deployed.
>翻译：ZooKeeper是用于维护配置信息，命名，提供分布式同步以及提供组服务的集中式服务。 所有这些类型的服务都以某种形式被分布式应用程序使用。 每次实施它们时，都会进行很多工作来修复不可避免的错误和竞争条件。 由于难以实现这类服务，因此应用程序最初通常会跳过它们，这会使它们在发生更改时变得脆弱并且难以管理。 即使部署正确，这些服务的不同实现也会导致管理复杂。

是一种分布式协调服务。分布式协调服务可以在分布式系统中共享配置，协调锁资源，提供命名服务。

#Zookeeper的数据模型
>ZooKeeper has a hierarchal name space, much like a distributed file system. The only difference is that each node in the namespace can have data associated with it as well as children. It is like having a file system that allows a file to also be a directory. Paths to nodes are always expressed as canonical, absolute, slash-separated paths; there are no relative reference. Any unicode character can be used in a path subject to the following constraints:  
>>The null character (\u0000) cannot be part of a path name. (This causes problems with the C binding.)
The following characters can't be used because they don't display well, or render in confusing ways: \u0001 - \u001F and \u007F
\u009F.
The following characters are not allowed: \ud800 - uF8FF, \uFFF0 - uFFFF.
The "." character can be used as part of another name, but "." and ".." cannot alone be used to indicate a node along a path, because ZooKeeper doesn't use relative paths. The following would be invalid: "/a/b/./c" or "/a/b/../c".
The token "zookeeper" is reserved.

>翻译：ZooKeeper有一个分层命名空间，很像分布式文件系统。唯一不同的是，命名空间中的每个节点都可以和子节点一样有数据关联。这就像拥有一个文件系统，允许一个文件也可以是一个目录。 节点的路径始终表示为标准的、斜杠分隔的绝对路径，没有相对路径。 任何Unicode字符都可以在路径中使用，但要受以下约束：
>>空字符 (\u0000) 不能成为路径名的一部分。(这将导致 C 绑定的问题)。
下面的字符不能使用，因为它们不能很好地显示，或者会造成混乱：\u0001 - \u001F和 \u007F。
\u009F。
不允许使用以下字符。\uF800 - uF8FF，uFFF0 - uFFFF。
". "字符可以作为另一个名称的一部分，但是". "和".."不能单独用来表示路径上的节点，因为ZooKeeper不使用相对路径。下面的内容是无效的。"/a/b/./c "或"/a/b/.../c"。
“ zookeeper”被保留。

可见，zookeeper数据模型很像树和文件目录的结构，Zookeeper的数据存储也同样是基于节点，这种节点叫做Znode。

>### ZNodes
>Every node in a ZooKeeper tree is referred to as a znode. Znodes maintain a stat structure that includes version numbers for data changes, acl changes. The stat structure also has timestamps. The version number, together with the timestamp, allows ZooKeeper to validate the cache and to coordinate updates. Each time a znode's data changes, the version number increases. For instance, whenever a client retrieves data, it also receives the version of the data. And when a client performs an update or a delete, it must supply the version of the data of the znode it is changing. If the version it supplies doesn't match the actual version of the data, the update will fail. (This behavior can be overridden. )
>翻译：ZooKeeper树中的每个节点都被称为znode。znode维护一个stat结构，其中包括数据变化的版本号、acl变化的版本号。这个stat结构也有时间戳。版本号和时间戳一起，允许ZooKeeper验证缓存并协调更新。每当znode的数据发生变化，版本号就会增加。例如，每当客户端检索数据时，它也会收到数据的版本号。而当一个客户端执行更新或删除时，它必须提供它要改变的znode的数据版本。如果它提供的版本与数据的实际版本不匹配，更新将失败。(这个行为可以被覆盖)
>#### Watches
>Clients can set watches on znodes. Changes to that znode trigger the watch and then clear the watch. When a watch triggers, ZooKeeper sends the client a notification. More information about watches can be found in the section [ZooKeeper Watches](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#ch_zkWatches).
>翻译：客户端可以在znode上设置Watch。对该znode的更改会触发Watch，然后清除Watch。当Watch触发后，ZooKeeper会向客户端发送一个通知。
>#### Data Access
>The data stored at each znode in a namespace is read and written atomically. Reads get all the data bytes associated with a znode and a write replaces all the data. Each node has an Access Control List (ACL) that restricts who can do what.
>翻译：存储在命名空间中每个znode的数据是以原子方式读和写的。 读取将获取与znode关联的所有数据字节，而写入将替换所有数据。每个节点都有一个访问控制列表(ACL)，限制谁可以做什么。
>ZooKeeper was not designed to be a general database or large object store. Instead, it manages coordination data. This data can come in the form of configuration, status information, rendezvous, etc. A common property of the various forms of coordination data is that they are relatively small: measured in kilobytes. The ZooKeeper client and the server implementations have sanity checks to ensure that znodes have less than 1M of data, but the data should be much less than that on average. Operating on relatively large data sizes will cause some operations to take much more time than others and will affect the latencies of some operations because of the extra time needed to move more data over the network and onto storage media. If large data storage is needed, the usually pattern of dealing with such data is to store it on a bulk storage system, such as NFS or HDFS, and store pointers to the storage locations in ZooKeeper
>翻译：ZooKeeper不是被设计成一个通用数据库或大型对象存储库。相反，它管理的是协调数据。这些数据可以以配置、状态信息、状态信息、交会等形式出现。各种形式的协调数据的一个共同属性是它们相对较小：以千字节为单位来衡量。ZooKeeper客户端和服务器的实现都有理智检查，确保znode的数据量小于1M，但平均下来，数据量应该远远小于这个量。在相对较大的数据量上操作，会导致某些操作所需的时间比其他操作要长得多，并且会影响到某些操作的延迟，因为需要额外的时间来将更多的数据通过网络移动到存储介质上。如果需要大数据存储，通常处理这类数据的模式是将其存储在批量存储系统中，如NFS或HDFS，并在ZooKeeper中存储指向存储位置的指针。
>#### Ephemeral Nodes
>ZooKeeper also has the notion of ephemeral nodes. These znodes exists as long as the session that created the znode is active. When the session ends the znode is deleted. Because of this behavior ephemeral znodes are not allowed to have children.
>翻译：ZooKeeper还具有临时节点的概念。 只要创建znode的会话处于活跃状态，这些znode就存在。 会话结束时，将删除znode。 由于这种特性，所以临时节点不允许有孩子节点。
  >#### Sequence Nodes -- Unique Naming
>When creating a znode you can also request that ZooKeeper append a monotonically increasing counter to the end of path. This counter is unique to the parent znode. The counter has a format of %010d -- that is 10 digits with 0 (zero) padding (the counter is formatted in this way to simplify sorting), i.e. "0000000001". Note: the counter used to store the next sequence number is a signed int (4bytes) maintained by the parent node, the counter will overflow when incremented beyond 2147483647 (resulting in a name "2147483648").
>翻译：创建znode时，您还可以要求ZooKeeper在路径末尾附加一个单调递增的计数器。这个计数器是父znode独有的。计数器的格式是 %010d ----也就是 10 位数字带 0 (0) 填充 (计数器的格式是为了简化排序)，即 "000000000001"。注意：用于存储下一个序列号的计数器是由父节点维护的有符号的int（4bytes），当增量超过2147483647时，计数器将溢出（导致名称为"-2147483648"）。


Znode包含以下结构：  
   - data：  
  Znode存储的数据信息。
   - ACL：
记录Znode的访问权限，即哪些人或哪些IP可以访问本节点。
   - stat：
包含Znode的各种元数据，比如事务ID、版本号、时间戳、大小等等。
   - child：
当前节点的子节点引用，类似于二叉树的左孩子右孩子。
另：Zookeeper是为读多写少的场景所设计。Znode并不是用来存储大规模业务数据，而是用于存储少量的状态和配置信息，每个节点的数据最大不能超过1MB。

# Zookeeper集群
Zookeeper Service集群是一主多从结构。在更新数据时，首先更新到主节点（这里的节点是指服务器，不是Znode），再同步到从节点。在读取数据时，直接读取任意从节点。为了保证主从节点的数据一致性，Zookeeper采用了ZAB协议，这种协议非常类似于一致性算法Paxos和Raft。

关于ZAB:
1. ZAB协议定义的三种状态
    - Looking：选举状态
    - Following：Follower（从节点）所处的状态。
    - Leading：Leader（主节点）所处的状态
2. ZXID
  节点本地的最新事务编号，包含epoch和计数两部分。
3. 崩溃恢复过程（待更新）

# 应用
1. 分布式锁
2. 服务注册和发现
3. 共享配置和状态信息

参考：[程序员小灰：《漫画：什么是ZooKeeper？》](https://juejin.im/post/5b037d5c518825426e024473)
