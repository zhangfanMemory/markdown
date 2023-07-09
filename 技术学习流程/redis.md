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