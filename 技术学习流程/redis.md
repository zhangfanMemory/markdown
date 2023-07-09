# redis

## redis对象头
``` java
struct RedisObject {
 int4 type; // 4bits
 int4 encoding; // 4bits
 int24 lru; // 24bits
 int32 refcount; // 4bytes
 void *ptr; // 8bytes，64-bit system
}
``` 
1. 不同的对象具有不同的类型 type(4bit)，
2. 同一个类型的 type 会有不同的存储形式encoding(4bit)
3. lru属性：缓存淘汰策略；
   1.  Redis 通过维护一个全局的 LRU 时钟来跟踪对象的最近访问时间。当一个对象被访问时，Redis 会更新该对象的 lru 属性，将其设置为当前的 LRU 时钟值
   2.  当 Redis 需要回收内存或进行淘汰时，它会选择 lru 属性值最小（最近最少使用）的对象作为淘汰的候选对象
   3.  lru 属性只是 Redis 内部使用的属性，并不直接暴露给 Redis 用户
4. refcount ： 记录对象的引用计数（jvm引用计数法？）
5. *ptr：用于指向实际数据的指针

## 数据结构

1. 数据结构
   1. String
      1. redis的字符串是以字节数组形式存在的，是可被修改的字符串；redis对字符串的存储有单独的数据结构
      2. **数据结构**叫：SDS："Simple Dynamic String" 的中文翻译是"简单动态字符串" 于存储和处理可变长度的字符串数据
         1. ；
         ```java 
         struct SDS<T> {
                T capacity; // 数组容量
                T len; // 数组长度
                byte flags; // 特殊标识位，不理睬它
                byte[] content; // 数组内容
            }
        1. ![](/技术学习流程/pic/2023-07-07-16-08-54.png)
        2. 可修改的字符串所以支持append操作
           1. 扩容方式先拓展，再复制，再append
           2. 字符串在长度小于 1M 之前，扩容空间采用加倍策略，也就是保留 100% 的冗余空间
           3. 当超过1m以后为了节省空间，每次增加空间1m
        3. sds 使用范性支持short或是byte类型空间极致压缩
        4. Redis 规定字符串的长度不得超过 512M 字节
        5. 字符串的 **存储方式** 两种emb / raw
           1. 如果字符串长度超过44个字符就用raw否则emb
      3. 操作：
         1. ```java 
            > set name codehole 
            OK 
            > get name 
            "codehole"
            > exists name 
            (integer) 1 
            > del name 
            (integer) 1 
            > get name 
            (nil);  
            ``` 
         2. mget 合并返回
         3. setex name 5 zhangfan ：5s 过期删除key为name ，value为zhangfan的值
         4. setnx name zhangfan ： name不存在就创建name为zhangfan的值
   2. list
      1. 异步队列实现
      2. rpush books python java golang 从右边放入
      3. rpop / lpop 可以从左右提取
      4. 消息对列实现，可以用blpop，阻塞的方式拉数据
   3. hash
      1.  hset books java "think in java"
      2.  hgetall books # entries()，key 和 value 间隔出现
      3.  hget books java -> "think in java"
   4. set
      1.  sadd books python
      2.  spop books # 弹出一个 -- > python
   5. zset
      1.  zadd books 9.0 "think in java"
      2.  zrange books 0 -1 # 按 score 排序列出，参数区间为排名范围
      3. zset 是一个复合结构，一方面它需要一个 hash 结构来存储 value 和 score 的对应关系，另一方面需要提供按照 score 来排序的功能，还需要能够指定 score 的范围来获取 value 列表的功能
   
### 压缩列表
概念：压缩列表是一块连续的内存空间，元素之间紧挨着存储，没有任何冗余空隙
zset 和 hash 容器对象在元素个数较少的时候，采用压缩列表 (ziplist) 进行存储
![](/技术学习流程/pic/2023-07-07-17-45-39.png)  


#### 数据结构
```java
struct ziplist<T> {
 int32 zlbytes; // 整个压缩列表占用字节数
 int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个
节点
 int16 zllength; // 元素个数
 T[] entries; // 元素内容列表，挨个挨个紧凑存储
 int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
}  
```

![](/技术学习流程/pic/2023-07-07-17-47-34.png)

entry 块随着容纳的元素类型不同，也会有不一样的结构。
``` java
struct entry {
 int<var> prevlen; // 前一个 entry 的字节长度;可以倒着遍历找到前一个元素
 int<var> encoding; // 元素类型编码，用于决定content中的编码格式
 optional byte[] content; // 元素内容
}  
```

理解：他是一块连续的内存空间频繁的扩容修改不太适合，涉及到扩容及复制

### 快速列表
早期版本存储 list 列表数据结构使用的是压缩列表 ziplist 和普通的双向链表linkedlist，也就是元素少时用 ziplist，元素多时用 linkedlist

概念：quicklist 是 压缩列表 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来
![](/技术学习流程/pic/2023-07-07-18-02-39.png)

``` java
struct quicklistNode {
 quicklistNode* prev;
 quicklistNode* next;
 ziplist* zl; // 指向压缩列表
 int32 size; // ziplist 的字节总数
 int16 count; // ziplist 中的元素数量
 int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
}
``` 
``` java
struct quicklist {
 quicklistNode* head;
 quicklistNode* tail;
 long count; // 元素总数
 int nodes; // ziplist 节点的个数
 int compressDepth; // LZF 算法压缩深度
 ...
}
``` 
快速列表中的ziplist 默认单个长度为8k，超过8k会将ziplist压缩为 ziplist_compressed
ziplist_compressed ：
1. 是一种压缩编码格式用于在有效区间内高效存储**zset和hash**数据
2. ziplist用于存储紧凑连续存储数据结构，用于存储较小的zset和hash，存储量数据变大就会变成compress，在快表中默认为8k

### 跳表
首先理解Zset的实现：内部实现是一个 hash 字典加一个跳跃列表 (skiplist)
![](/技术学习流程/pic/2023-07-08-12-07-51.png)

#### 基本结构
Redis 的跳跃表共有 64 层 如图只有上下四层，可以存储 2^64 次方个元素
![示意图](/技术学习流程/pic/2023-07-08-12-09-11.png)
kv由zslnode构成,kv之间通过双向链表链接，同一层的kv通过指针串联

```java
struct zslnode {
 string value;
 double score;
 zslnode*[] forwards; // 多层连接指针
 zslnode* backward; // 回溯指针
}
```

#### 查找过程
1. 如果只有一层，那么寻找元素，时间将是o(n)；跳表与二分查找类似，可以一层层向下递归，查找时间复杂度为o(logn)
2. 搜索路径：从最顶层到最底层找到期望节点的路径为搜索路径
3. 节点高度：前面说过跳表最高为64层，那如果要插入新的元素，该元素的高度怎么判断
   1.  随机分配机制：50% 的 Level1，25% 的 Level2，12.5% 的 Level3，一直到最顶层 2^-63，因为这里每一层的晋升概率是 50%
   2.  插入：通过搜索路径找到节点在最底层的位置，通过随机分配测策略是否上升到上层节点，这样新节点就可以参与到更高级的搜索节点中
   3.  删除：通过搜索路径找到节点，若要删除，需要删除该节点在所有层级中的节点，
   4.  更新：先删除再插入
   
#### 总结：
1. 对于内存的占用：跳跃列表的每个节点都需要额外的指针来构建层级结构，因此占用的内存空间相对较大。对内存比较敏感
2. 对跳表节点的更新麻烦：
   1. 按照搜索路径找到节点
   2. 更新节点的值
   3. 更新上层链表的指针：遍历该节点的上层节点，修改上层链表的前后指针
   4. 如果涉及到层级的变更，那么需要相应修改该层级上的相关节点的前后指针
3. 但是因为读写操作远大于写操作，所以依然抗用

### 基数树 rax
Redis 五大基础数据结构里面，能作为字典使用的有 hash 和 zset。
1. hash 不具备排序功能，
2. zset 则是按照 score 进行排序的。
3. rax 跟 zset 的不同在于它是按照 key 进行排序的。
   ![](/技术学习流程/pic/2023-07-08-14-48-29.png)

```java
struct raxNode {
 int<1> isKey; // 是否没有 key，没有 key 的是根节点
 int<1> isNull; // 是否没有对应的 value，无意义的中间节点
 int<1> isCompressed; // 是否压缩存储，这个压缩的概念比较特别
 int<29> size; // 子节点的数量或者是压缩字符串的长度 (isCompressed)
 byte[] data; // ***路由键***、子节点指针、value 都在这里
}
```
这个树：如果一个中间节点有多个子节点，那么路由键就只是一个字符。如果只有一个子节点，那么路由键就是一个字符串。后者就是所谓的「压缩」形式
![](/技术学习流程/pic/2023-07-08-14-55-13.png)

1. 非压缩节点 如果子节点有多个，那就不是压缩结构，存在多个路由键，一个键是一个字符。
   1. 像es有两个儿子
   2. 而aster只有一个儿子
2. 如果是叶子节点，child 字段就不存在

![路由键值对es/aster](/技术学习流程/pic/2023-07-08-14-57-27.png)

## redis的线程io模型

### 前提
1. Redis 是个单线程程序
2. 所有的运算都是内存级别的运算
3. Redis 单线程如何处理那么多的并发客户端连接？
   1. reactor线程模型 selector，理解reactor

## 管道
(Pipeline) 本身并不是 Redis 服务器直接提供的技术，这个技术本质上是由客户端提供的
实际上理解就是合并操作
1. 读写读写 两次网络传输
2. 读读写写 -> 一次网络传输

## 数据持久化
1. RDB  ：redis databse
   1. 简单明了快照方式存储，存储的磁盘文件是2进制文件
   2. 有save和bgsave两种方式，sava直接停止进行数据快照存储，bgsave通过fork子进程进行存储
   3. fork是调用c库glibc的函数创建一个子进程进行数据存储处理
   4. 创建时子进程与父进程内存数据保持一致
   5. 当有客户端提交数据时候，父进程会采用copy on right的方式进行数据读取，将数据复制一份进行修改，修改后的数据存储在父进程中当存储完成最后进行修改源文件
   6. 子进程的数据在诞生那一刻就不会进行修改
   7. RDB的特点
      1. 包含着数据库完整的状态，可在数据库重启时候进行恢复
      2. 紧凑和高效的：rdb是二进制文件，有紧凑的格式和高效的存储性能
      3. 可手动触发，定时触发以及写入触发
2. AOF: Append-Only File
   1. AOF 日志存储的是 Redis 服务器的顺序指令序列，**AOF 日志只记录对内存进行修改的指令记录。**
   2. 假设 AOF 日志记录了自 Redis 实例创建以来所有的修改性指令序列，那么就可以通过对一个空的 Redis 实例顺序执行所有的指令，也就是「重放」，来恢复 Redis 当前实例的内存数据结构的状态。
   3. aof是将命令添加到aof文件后
   4. 但是当更改命令很多的时候，会造成aof文件特别的大，这个时候需要对aof文件进行缩水，减少占用空间
      1. 使用命令bgrewriteaof，同样开辟一个子进程进行修改
      2. 遍历当前数据库的kv对，重写aof文件，保存最少的更新修改命令
      3. 重写aof后将，这期间产生的命令添加到aof文件后面
   5. aof是将指令先写到内存文件中去，然后通过操作系统的定时刷新到磁盘上
      1. aof写到内存但是没刷到磁盘，数据库重启数据丢失
      2. 利用操作系统的**fsync** 命令强制将文件数据刷到磁盘上
         1. 可以指定周期1s一次/
         2. 不执行（利用操作系统自动执行）/
         3. 来一次修改命令就执行一次
3. 关于aof和RDB文件是不是都是二进制文件
   1. 是的
   2. aof文件主要是修改redis库的命令语句，但是包含redis的数据结构和编码信息所以用二进制进行存储
   3. rdb是纯粹的二进制文件，只存储键值对，更加的紧凑
4. 两者区别联系
   1. rdb文件通常比aof小，一是只存kv一是紧凑二进制
   2. rdb恢复较快，aof要逐条遍历命令恢复慢
   3. aof恢复完整，rdb因为是快照所以恢复可能漏
      1. 现在一般都是rdb然后追加aof的形式。（混合持久化）
      2. ![](/技术学习流程/pic/2023-07-09-17-09-08.png)
      3. 这里的 AOF 日志不再是全量的日志，而是自持久化开始到持久化结束的这段时间发生的增量 AOF 日志，通常这部分 AOF 日志很小

## 数据同步
1. redis 符合最终一致不符合强一致性；异步同步数据
   1. 但是redis3.0支持write命令，将异步改为同步
      1. set zhang fan
      2. write 1 0 （1：一个slave. 0 ms等待时间，若为0则无限等待）
2. 主备节点同步方式
   1. 增量同步
      1. 就是一个环形的buffer，主节点将新增命令写到环形buffer中，异步将buffer中的数据发给slave
      2. 问题如果满了，buffer会覆盖掉之前的数据，咋办：有了全量快照同步
   2. 快照同步（全量）
      1. slave更不上master了，master将自己的数据进行快照备份，将快照写到buffer。
      2. slave首先同步rdb文件，然后同步快照
      3. 还是会出现，slave还没同步完rdb结果master的buffer有被重复覆盖了，反复反复
      4. ---- > 合理设计buffer数组
      5. ![](/技术学习流程/pic/2023-07-09-17-39-20.png)  


## 多节点redis

### sentinel 哨兵
1. 一个高可用集群来监控主节点是否异常，并在异常的时候切换主备节点
2. ![](/技术学习流程/pic/2023-07-09-18-46-56.png)
3. redis采用的是主从同步的方式可能会数据丢失，sentinel模式下对此采用的方案是提供两个参数
   1. min-slaves-to-write 1 ： 至少有一个节点在进行正常的复制，否则不对外提供写相应，redis失去可用性
   2. min-slaves-max-lag 10 ： 在10s内slave有与哨兵进行节点反馈

### cluster 集群
1. ![](/技术学习流程/pic/2023-07-09-18-51-25.png)
2. 去中心化的三个节点，分别存储不同的数据
3. Cluster 将所有数据划分为 16384 的 slots；每个节点负责其中一部分槽位
4. 槽位定位算法
   1. Cluster 默认会对 key 值使用 crc32 算法进行 hash 得到一个整数值，然后用这个整数值对 16384 进行取模来得到具体槽位。
   2. redis的键值对是如何部署到各个槽中的
      1. 将键进行CRC16hash算法计算成一个hash值，然后对该hash惊醒16384的取余，该键就找到了对应的slot
   3. 通常情况下，集群上的各个redis节点也具备主备节点
      1. 主备节点切换是通过gossip 病毒传播协议类似~~ 超过半数节点任务你下线了你就是下线了，cluster进行主备切换
   4. 当有新的redis加入cluster或者删除了某个cluster
      1. 会从已有的redis节点中均匀分出部分slot给该节点
      2. 然后进行数据迁移
      3. 在迁移过程中此时由client进行访问： 
         1. 老节点给你一个moved，让你重新访问，会告诉你去访问那个节点
         2. client带着asking指令重新访问数据迁移到的节点
         3. ![](/技术学习流程/pic/2023-07-09-19-13-33.png)

### codis 中国人研发集群


## 过期策略
对过期key的处理
1. 定时删除
   1. 将设置了expire的key进行单独hash表存储
   2. 定时扫描策略
      1. 从字典中找到20个key
      2. 删除这20个key中已经超时的key
      3. 如果删除比例超过1/4则继续筛选
   3. 由此可以看出不要把过期时间设置在一块
   4. 从库没有过期策略，主库在key过期回新增del语句在aof文件上，从库跟着删除，会出现数据不一致场景
2. LRU机制
   1. 实际内存超出最大内存、
3. (maxmemory-policy) 最大内存淘汰机制
   1. noeviction 不会继续服务写请求 (DEL 请求可以继续服务)，读请求可以继续进行
   2. volatile-lru 尝试淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰
   3. volatile-ttl 剩余寿命最小的（剩余时间最小的）
   4. volatile-random 不过淘汰的 key 是过期 key 集合中随机的 key（设置随机时间的随机淘汰）
   5. allkeys-lru 最近最少使用的key，包含没有设置过期时间
   6. allkeys-random 随机淘汰所有的
4. 惰性删除