# mybatis学习

## 缓存
![](/技术学习流程/pic/2023-07-02-18-47-53.png)
1. 第一次发出一个查询 sql，sql 查询结果写入 sqlsession 的一级缓存中，缓存使用的数据结构是一个 map。key：MapperID+offset+limit+Sql+所有的入参
2. 如果两次中间出现 commit 操作
（修改、添加、删除），本 sqlsession 中的一级缓存区域全部清空，下次再去缓存中查询不到所
以要从数据库查询，从数据库查询到再写入缓存。

1. mybatis 的二级缓存是通过 CacheExecutor 实现的。mapper 以命名空间为单位创建缓存数据结构，结构是 map
   1. Mybatis 全局配置中启用二级缓存配置
   2. 在对应的 Mapper.xml 中配置 cache 节点
   3. 在对应的 select 查询节点中添加 useCache=true

问题：同时存在1/2级缓存是怎么运行的
1. 当执行一个查询语句时，MyBatis首先检查一级缓存，看是否有缓存的结果。
2. 先查询二级缓存，然后查询一级缓存
3. 如果配置了二级缓存且查询的结果在二级缓存中存在，则从二级缓存中获取结果，并返回。
4. 如果缓存中没有结果，则执行实际的SQL查询操作，获取数据库的查询结果。
5. 将查询结果存储到一级缓存和二级缓存中，以便下次使用。
6. 
**需要注意的是，对于更新、插入、删除等会修改数据库的操作，MyBatis会清空一级缓存和相关的二级缓存，以保证数据的一致性。**

## sqlsession 和 statement的区别
1. SqlSession 是 MyBatis 框架中用于与**数据库交互的主要接口**。它是一个会话对象，用于管理数据库**连接并执行 SQL 语句**
2. Statement 是 Java JDBC API 中的一个接口，它表示了一个预编译的 SQL 语句。**在 JDBC 中，Statement 用于执行 SQL **语句并返回结果。
3. SqlSession 是 MyBatis 用于管理数据库连接和执行 SQL 操作的会话对象，它封装了底层的 Statement 实现
4. 对应关系是一对一的关系

## mybtais 执行sql 的具体流程
1. 读取mybatis-congig.xml（mybatis的配置文件） 和 mapper.xml（映射文件，定义了java与sql语句对应关系）
2. 生成sqsessionFactory（主要用于生成sqlsession）
3. 执行sqsessionFactory.openSession(),开启一个sqlsession
4. 解析映射文件，在创建sqlsession时候会将java和sql语句映射关系存在内存中
5. 获取mapper的接口代理对象，通过**sqlsession.getmapper获取代理对象**
6. **通过 Mapper接口的代理对象调用相应的方法，即可执行 SQL语句，MyBatis 会根据方法名称和参数类型来找到对应的映射SQL，并执行它； 在执行 SQL 时，MyBatis 会创建相应的 Statement 对象（可以是 Statement、PreparedStatement 或 CallableStatement），并将参数设置到 Statement 中。**
7. 执行完 SQL 后，MyBatis 会根据映射文件中的配置将查询结果映射到 Java 对象中。这样，就可以将查询结果赋值给 Java 对象，形成最终的结果对象。
8. 返回结果
9. 删除sqlsession

## mybatis中如何通过sqlsession.getmapper得到代理对象
mybatis的mapper代理对象如何生成的
1. MyBatis会为每个Mapper接口中的方法创建一个对应的MapperMethod对象。MapperMethod中包含了SQL语句的执行逻辑和结果映射等信息。
2. 生成的代理对象实际上是MapperProxy的实例。MapperProxy是MyBatis中用于代理Mapper接口的核心类，它实现了Java的InvocationHandler接口
3. 当代理对象的方法被调用时，实际上是调用了MapperProxy的invoke()方法。
4. 会根据调用的方法名称和参数类型，在Configuration对象中查找对应的MapperMethod对象。然后，invoke()方法会执行MapperMethod中定义的SQL语句，并将执行结果返回给调用者。


## 随笔
每当我们使用 MyBatis 开启一次和数据库的会话，MyBatis 都会创建出一个 SqlSession 对象，表示与数据库的一次会话，而每个 SqlSession 都会创建一个 Executor 对象

如下图所示，MyBatis 的一次会话：在一个 SqlSession 会话对象中创建一个localCache本地缓存，对于每一次查询，都会根据查询条件尝试去localCache本地缓存中获取缓存数据，如果存在，就直接从缓存中取出数据然后返回给用户，否则访问数据库进行查询，将查询结果存入缓存并返回给用户（如果设置的缓存区域为STATEMENT，默认为SESSION，在一次会话中所有查询执行后会清空当前 SqlSession 会话中的localCache本地缓存，相当于“关闭”了一级缓存）

![](/技术学习流程/pic/2023-11-24-11-51-24.png)