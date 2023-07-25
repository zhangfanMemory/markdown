# spring 

## 容器
1. 代码：Spring 容器就是一个实现了 ApplicationContext 接口的对象
2. 业务：用来管理对象的 ： 容器将创建的对象，把它们连接在一起，配置它们，并管理它们的整个生命周期从创建到销毁
3. 对象实例化，只是得到一个对象，还不是一个完全的Spring中的bean，我们实例化后的这个对象还没有完成依赖注入，没有走完一系列的生命周期
**Spring官网上说到，在Spring实例化一个对象有三种方式：**
构造函数
实例化工厂方法
静态工厂方法

### spring 特征
spring：
![](/技术学习流程/pic/2023-07-01-19-15-08.png)
![](/技术学习流程/pic/2023-07-01-19-19-32.png)
![](/技术学习流程/pic/2023-07-01-19-22-25.png)

### 父子容器
HierarchicalBeanFactory 父子级联
父子级联 IoC 容器的接口，子容器可以通过接口方法访问父容器； 通过HierarchicalBeanFactory 接口， Spring 的 IoC 容器可以建立父子层级关联的容器体系，**子容器可以访问父容器中的 Bean，但父容器不能访问子容器的 Bean。**
Spring 使用父子容器实现了很多功能，比如在 Spring MVC 中，展现层 Bean 位于一个子容器中，而业务层和持久层的 Bean 位于父容器中。这样，展现层 Bean 就可以引用业务层和持久层的 Bean，而业务层和持久层的 Bean 则看不到展现层的 Bean。

## 类初始化流程
![](/技术学习流程/pic/2023-07-02-17-21-54.png)

### 三级缓存
1. singletonobject： 缓存最终的单例池结果
2. earlysingletonobject： 缓存的单例不完善，没有填充属性等，用于解决循环依赖场景
3. 三级缓存：主要用于切面场景：在aop场景中暴露出来的对象应该是代理对象而不是原对象，但是代理对象是初始化完成才进行生成，如果出现循环依赖场景就无法生成，此时就会用到三级缓存保存所有对象的动态代理配置信息，提前进行aop，生成动态代理
4. ![](/技术学习流程/pic/2023-07-21-17-43-42.png)

## 依赖注入
依赖注入属于ioc的具体实现，如何将bean交由spring管理，以及spring如何创建一个完整的bean  

### 三种方式
#### 属性注入
   1. java代码
   2. 
   ```java
    @RestController
    public class UserController {
        // 属性对象
        @Autowired
        private UserService userService;

        @RequestMapping("/add")
        public UserInfo add(String username, String password) {
            return userService.add(username, password);
        }
    }
```
问题：
1. 无法注入一个不可变的final对象
2. 只能适用于ioc容器，移植性不好
3. 过于简单的注入方式导致一个bean很臃肿，可能多出很多的无用注入，可能影响单一职责

#### 构造器注入

```java
@RestController
public class UserController {
    // 构造方法注入
    private UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @RequestMapping("/add")
    public UserInfo add(String username, String password) {
        return userService.add(username, password);
    }
}
```
特点：
1. 可注入不可变的对象final修饰的对象，如上述代码中的private final UserService userService;
2. 注入的对象不会被修改
   1. 通过构造方法注入，只有在对象创建的时候执行一次构造方法，注入bean
3. 注入对象完全初始化
   1. 在执行构造方法之前，需要注入的对象会被完全初始化
4. 移植性更好
    

#### setter注入

```java
@RestController
public class UserController {
    // Setter 注入
    private UserService userService;

    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    @RequestMapping("/add")
    public UserInfo add(String username, String password) {
        return userService.add(username, password);
    }
}
```
问题 ：
1. 不能注入不可变对象（final 修饰的对象）；
2. 注入的对象可被修改。
   1. Setter 注入提供了 setXXX 的方法，意味着你可以在任何时候、在任何地方，通过调用 setXXX 的方法来改变注入对象，所以 Setter 注入的问题是，被注入的对象可能随时被修改。

#### 注入不可变对象
依赖注入用构造器方式注入final修饰的对象
   1. 线程安全需求：在多线程场景下，为防止被注入的对象发生意外修改
