## ZK
**Zookeeper 是一个分布式协调服务，可用于服务发现，分布式锁，分布式领导选举，配置管理，注册中心，分布式通知（利用watch机制）等。**


leader: 支持写操作，然后同步给给各个节点（不包含obser），超过半数成功则写入成功
follower: 支持读操作，可以投票
observer: 支持读操作，不可以投票
follow和obeser 都是需要将写操作转发给leader， obser不支持投票新增是为了其延展性

## ZAB
 ZooKeeper Atomic Broadcast , ZooKeeper 原子消息广播协议

### ZXID ：事物id
64位的数字。 高32位为当前leader的epoch，低32位是事物id，从0开始递增

### epoch
每个当选产生一个新的 Leader 服务器，就会从这个 Leader 服务器上取出其本地日志中最大事务的 **ZXID**，并从中读取
epoch值，然后加 1，以此作为新的epoch
皇帝的权杖

### Zab 协议有两种模式-恢复模式（选主）、广播模式（同步）
1. 选举阶段：
   1. 节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准 leader。只有到达 广播阶段（broadcast） 准 leader 才会成为真正的 leader。这一阶段的目的是就是为了选出一个准 leader，然后进入下一个阶段
2. 发现阶段：
   1. ，followers 跟准 leader 进行通信，leader会同步 followers 最近接收的事务提议。这个一阶段的主要目的是**发现**当前大多数节点接收的**最新提议**，并且准 leader 生成新的 epoch，让 followers 接受，**更新它们的 accepted Epoch**
   2. 一个 follower 只会连接一个 leader，如果有一个节点 f 认为另一个 follower p 是 leader，f 在尝试连接 p 时会被拒绝，f 被拒绝之后，就会进入重新选举阶段
3. 同步阶段：
   1. 同步阶段主要是利用 leader 前一阶段获得的最新提议历史，同步集群中所有的副本。只有当 大多数节点都同步完成，准 leader 才会成为真正的 leader。follower 只会接收 zxid 比自己的 lastZxid 大的提议
4. 广播阶段：
   1. ：到了这个阶段，Zookeeper 集群才能正式对外提供事务服务，并且 leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步
**提交事务并不像 2PC 一样需要全部 follower 都 ACK，只需要得到超过半数的节点的 ACK 就可以了。**

#### 投票机制：
每个 sever 首先给自己投票，然后用自己的选票和其他 sever 选票对比，权重大的胜出，使用权重较大的更新自身选票箱。
1. 每个 Server 启动以后都询问其它的 Server 它要投票给谁
2. 收到所有 Server 回复以后，就计算出 zxid 最大的哪个 Server，zxid相等则判断机器id大的获胜
3. 更新自己的投票箱
4. 得票超过半数获胜

具体例子：
1. 服务器 1 启动，给自己投票，然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，服务器 1 的状态一直属于 Looking
2. 服务器 2 启动，给自己投票，同时与之前启动的服务器 1 交换结果，由于服务器 2 的编号大于1服务器 2 胜出，但此时投票数没有大于半数，所以两个服务器的状态依然是LOOKING
3. 此时服务器1，2的投票箱都是投给2的
4. 服务器 3 启动，给自己投票，同时与之前启动的服务器 1,2 交换信息，由于服务器 3 的编号最大所以服务器 3 胜出，此时投票数正好大于半数，所以服务器 3 成为领导者，服务器1,2 成为小弟。
5. 此时1，2，3的投票箱都是给3的，超过半数3是leader开始发现和同步阶段
6. 服务器 4 启动加入3的领导
7. 服务器 5 启动加入3的领导
8. 新加入的节点是以恢复模式的身份加入，先寻找leader，找到后进行同步阶段，同步zxid和epoch，再进行广播阶段

**广播模式需要保证 proposal 被按顺序处理，因此 zk 采用了递增的事务 id 号(zxid)来保证。所有的提议(proposal)都在被提出的时候加上了 zxid。**

## ZK的四种节点
1. PERSISTENT 持久节点
   1. 集群配置信息
   2. 服务注册信息
2. PERSISTENT_SEQUENTIAL：持久化顺序节点。
   1. 有序任务分配：每个任务请求都会创建一个持久顺序节点，任务处理器可以按照节点的顺序来依次处理任务，从而保证任务的有序性
   2. 有序事件通知
3. EPHEMERAL：暂时的节点。
   1. 临时会话状态信息：在分布式会话管理中，可以使用临时节点来存储会话的状态信息，如登录状态、在线状态等
   2. 分布式锁：多个客户端可以创建同一个临时节点来竞争获取锁，只有一个客户端能够成功创建临时节点，从而获得锁的控制权
4. EPHEMERAL_SEQUENTIAL：暂时化顺序编号目录节点。
   1. 临时顺序节点常用于实现分布式队列的功能，每个客户端可以创建一个临时顺序节点，按照节点的顺序来加入队列。其他客户端可以按照相同的顺序来从队列中获取任务或消息进行处理

## watch机制:发布订阅
1. 当你在ZooKeeper上注册一个Watcher，它会在节点状态发生变化时得到通知，然后你可以在Watcher中处理相应的逻辑
2. ZooKeeper的Watcher是一次性的，一旦被触发，就需要重新注册。因此，当你在处理完Watcher的事件后，如果希望继续监视节点的变化，需要再次注册Watcher。
3. java需要使用Curator
4. 
``` java
CuratorFramework client = CuratorFrameworkFactory.newClient(ZK_CONNECTION_STRING,
                new ExponentialBackoffRetry(1000, 3));
client.start();
NodeCache nodeCache = new NodeCache(client, ZK_NODE_PATH);
  // 注册Watcher监听器
        nodeCache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                byte[] data = nodeCache.getCurrentData().getData();
                System.out.println("Node changed: " + new String(data));
            }
        });
``` 
