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

## ThreadLocal内存溢出问题：
1. 我们可以发现 get 和set 方法都可能触发清理方法expungeStaleEntry()，所以正常情况下是不会有内存溢出的。
2. 是如果我们没有调用get和set的时候就会可能面临着内存溢出。
3. 就算我们没有调用get和set和remove方法，线程结束的时候，也就没有强引用再指向ThreadLocal中的ThreadLocalMap了，这样ThreadLocalMap和里面的元素也会被回收掉。
4. 但是有一种危险是，如果线程是线程池的，在线程执行完代码的时候并没有结束，只是归还给线程池，这个时候ThreadLocalMap和里面的元素是不会回收掉的。