# THREADLOCAL
![](/技术学习流程/pic/2023-07-30-19-33-16.png)
1. 理解：每个线程会持有threadlocalMap；
2. 可以创建多个threadlocal对象
3. threadlocalmap是一个entry的数组
   1. ![](/技术学习流程/pic/2023-07-30-19-41-27.png)
   2. ![](/技术学习流程/pic/2023-07-30-19-43-26.png)
   3. 根据当前线程找到当前现成的threadlocalma的entry的数组
4. entry<THREADLOCAL-WEAK引用， 存的值>
   1. ![](/技术学习流程/pic/2023-07-30-19-44-30.png)
   2. KEY:threadlocal对象的弱引用
   3. value：你存的值   
5.      Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);

## ThreadLocal内存溢出问题：
1. 我们可以发现 get 和set 方法都可能触发清理方法expungeStaleEntry()，所以正常情况下是不会有内存溢出的。
2. 是如果我们没有调用get和set的时候就会可能面临着内存溢出。
3. 就算我们没有调用get和set和remove方法，线程结束的时候，也就没有强引用再指向ThreadLocal中的ThreadLocalMap了，这样ThreadLocalMap和里面的元素也会被回收掉。
4. 但是有一种危险是，如果线程是线程池的，在线程执行完代码的时候并没有结束，只是归还给线程池，这个时候ThreadLocalMap和里面的元素是不会回收掉的。


## 整理
1. 如果对象不持有threadlocal的没有强引用
2. 那么threadlocalMap中的key作为threadlocal作为弱引用就会被回收，那么出现entry为 null - value 状态，这种状态在getsetremove状态下会清理
3. 在threadlocalmap的get set 会进行判断，当map中key为null的话就会进行删除（expungeStaleEntry（））方法
4. ThreadLocal内存泄漏的根源是 ：由于ThreadLocalMap的生命周期跟Thread一样长，而不是threadlocal，如果没有手动删除对应key就会导致内存泄漏，而不是因为弱引用。
5. 如果threadlocal对象的持有已经丢失，在没人调用getsetremove，他的value在threadlocalmap中被引用不会丢失，threadlocalmap与thread存活时间一样长，在线程池的状态下会容易复现