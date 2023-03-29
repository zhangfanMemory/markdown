# io学习流程

## 用户态 & 内核态

**内核态**：内核本质是一种软件，控制计算机的硬件资源，并提供上层环境
**用户态**：上层应用程序活动空间，应用程序的执行要依赖于内核提供的资源，（包括cpu资源，存储资源，io资源）
**两者关系**：用户态的程序要访问内核态的资源可以通过：系统调用
**系统调用** ： 通过库函数及shell脚本进行封装，屏蔽复杂的底层实现细节，对系统调用进行封装提供简单接口个用户
**从用户态到内核态的其他种系统调用**
    1. 系统调用
    2. 异常：缺页异常，被动切换
    3. 外设中断：外设完成用户请求以后，会通知cpu发送中断信号。cpu会暂停即将执行的命令进而处理外设中断请求

## 五种io网络模型

### 阻塞io（bloking io）

**举例子**： 内核空间 -> 线程A -> 线程B ->线程C （工作队列）

&ensp;&ensp;当线程A要进行网络传输创建socket语句的时候，此时由**文件管理系统**创建一个由其管理的socket对象，这个对象包含发送缓冲区，接收缓冲区，等待队列；其中**等待队列**指向所有等待socket事件的线程  

&ensp;&ensp; 当线程执行到recv的时候，那么线程A就会移动到等待队列之后，此时工作队列没有线程A,所以线程A就不会占用cpu，而cpu轮训执行也只会执行线程BC  

应用进程调用recvform，执行系统调用，直到有数据报被拷贝到应用进程才返回，否则进程一直处于阻塞状态。**此时会把cpu给交出去不会一直占用**

### 非阻塞IO（non-blocking IO）

用轮询的方式一直请求内核看数据有没有拷贝完，所以非阻塞io不会释放cpu会一直轮询，**造成cpu浪费**

### 信号驱动式IO（signal-driven IO）

应用程序使用套接字进行信号驱动I/O，通过**sigaction系统调用**安装一个信号处理函数，用户进程运行不阻塞，当数据准备好以后，内核会发出一个sigio信号给信号处理函数，信号处理函数调用io操作处理数据。
缺点：信号IO在有大量的io操作的时候，会因为信号队列的溢出导致没法通知

### 异步IO（asynchronous IO）

io多路复用是将**文件的句柄的状态**通知给用户线程，由用户自行读取数据处理数据，异步io是数据内核已经处理完了，并且**放在用户线程的指定的缓冲区域**，当用户线程得到通知后可以直接使用数据；

### 多路复用IO

概念1：多路分离函数select
    1. 解决非阻塞io轮训的问题
    2. 系统内核提供多路分离函数select
    3. 用户将socket注册到select上
    4. **阻塞等待**select的反馈
    5. 数据到达后，select被激活
    6. 用户发起read将数据从内核拷贝到用户空间处理
       1. 涉及问题：内核数据拷贝的问题
    7. 一个线程可以注册多个socket，可以用一个线程不断的调用select读取被激活的socket，从而达到一个线程读取多个socket场景（处理多个io）

 1. 但是还是阻塞的，阻塞在select函数，有没有什么办法让多路复用不被阻塞

引出概念2： reactor下的io多路复用
    1. 用户线程注册到reactor上，然后reactor负责调用内核的select函数检查socket
    2. 当有socket被激活后通知其对应的用户线程（回调
    3. 所以这样情况下用户线程不会被阻塞，因为用户线程要处理io的时候，数据已经到达了

 1. **优点：单个线程可以处理多个网络接连io，原理在于不再又应用程序监听链接，而是由内核替代应用程序监听文件描述符；**
 举例：
    1. 用户调用select，**整个进程会阻塞(这里需要看一下)**
    2. 内核会监听select负责的socket，当有一个socket的数准备好，select就会返回，
    3. 然后用户在进行read操作，将数据从内核到用户区

io多路复用中的select poll epoll函数介绍：
    1. 这些机制主要做的是一个任务是**监听多个描述符**，一旦某个描述符就绪，就通知对应的程序进行对应的读写操作
    2. 但是收到通知以后还是线程自己去内核进行读取，而异步io实现会把数据从内核考到用户空间

select :
    1. &ensp;&ensp;监视三类文件描述符：writefds；readfds；exceptsfds；
    2. &ensp;&ensp;调用select函数会阻塞
    3. &ensp;&ensp;当select返回遍历查看那个描述符操作完成
    4. **缺点**：最大只能监听1024个描述符

poll ：
    1. 与select的区别在于没有最大的描述符监听限制
    2. 同样准备完成后需要调用pollfd轮训获取就绪的描述符、
    3. 随着描述符监听的增加，效率会降低很多

epoll ：
    1. epoll_creat（）:告诉内核监听的数目有多大，不是限制只是内核初始分配的建议，但是内核会占用这些描述符，所以使用完了，必须要关闭掉
    2. epoll_ctl（）:  epfd : epoll_creat的返回值 ；op：三个宏来表示：添加，删除，修改 fd：要监听的fd（文件描述符）；epoll_event:告诉内核需要监听什么事
    3. epoll_wait（）：等待epfd上的io事件，最多返回maxevents个事件；
![函数](c:/Users/Raytine/Desktop/24932558-a9eb8f1d6b9589a1.webp)  

![rector](c:/Users/Raytine/Desktop/24932558-5ac7cd4205d569bd.jpg)

## 讲一下什么Reactor 和 Proactor

Reactor包含如下角色：

 1.  Handle 句柄： 用来标识socket连接或是打开文件；
 2.  Synchronous Event Demultiplexer：同步事件多路分解器：由操作系统内核实现的一个函数；**用于阻塞等待发生在句柄集合上的一个或多个事件；（如select/epoll；）**
 3.  Event Handler：**事件处理接口**，拥有io文件句柄（可以通过get_handle获取，以及对handle的操作handle_event(读/写)）
 4.  Concrete Event HandlerA：实现应用程序所提供的特定事件处理逻辑；
 5.  Reactor：反应器，定义一个接口，实现以下功能：
     1.  供应用程序注册和删除关注的事件句柄；
     2.  使用handle_events运行事件循环；
     3.  reactor不断调用Synchronous Event Demultiplexer的select函数，有文件句柄被激活，select就会返回，handle_events会调用与文件句柄关联的事件处理器handle_event进行处理
     
![Alt text](c:/Users/Raytine/Desktop/24932558-493d08e7dab8ecfd.jpg)

### io多路复用的流程方式：
    1. 通过Reactor的方式，可以将用户线程轮询IO操作状态的工作统一交给handle_events事件循环进行处理
    2. 用户线程注册事件处理器之后可以继续执行做其他的工作（异步）
    3. 而Reactor线程负责调用内核的select函数检查socket状态。
    4. 当有socket被激活时，则通知相应的用户线程（或执行用户线程的回调函数）执行handle_event进行数据读取、处理的工作。[label](c:/Users/Raytine/Desktop/1.JPG%0D) ![Alt text](c:/Users/Raytine/Desktop/24932558-493d08e7dab8ecfd.jpg)
**因为select系统调用是阻塞的，所以io多路复用并不是真正的异步，只能称为异步阻塞io**

### 单线程模式异步非阻塞io
[label](c:/Users/Raytine/Desktop/1.JPG%0D) ![Alt text](c:/Users/Raytine/Desktop/24932558-493d08e7dab8ecfd.jpg)
acceptor收到了客户端的tcp的链接。建立成功后，通过dispatch将对应的bytebuf分发到指定的handler上，进行消息解码
    问题：  
    1. 一个nio线程无法处理太多的链接，即便cpu满负荷也不可以
     2. nio负载过重，处理速度会变慢
     3. 一旦nio线程跑飞，整个系统的通信模块都会变得不可用

### Reactor多线程模型
![Alt text](c:/Users/Raytine/Desktop/24932558-7b509f3abebe5eed.jpg)
不同处：
1. acceptor作为一个单独的nio线程监听服务端，接收client的request
2. 网络io操作读写由一个nio线程池负责，一个队列和多个可用的线程，这些线程负责消息的编解码及发送
3. 一个nio线程可以同时处理多个链路，但是一个链路只对应一个nio线程


### 主从Reactor多线程模型
![Alt text](c:/Users/Raytine/Desktop/24932558-8ad707941af96b22.jpg)
最大的特点就是接收客户端连接的不是一个单独的nio操作而是变成了一个独立的nio线程池
   1. 这个接收线程池收到请求并处理完以后会创建一个新的socketchannel然后注册到sub线程池的io上
   2. 然后由sub线程池中的对socketchannel中的数据进行编解码操作

**总结**：acceptor线程池仅仅用于客户端的登录握手等安全认证，建立成功以后就将链路注册到后端的subReactor线程池的io线程上，由io线程负责后续的io操作


Proactor：
