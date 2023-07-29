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
这个图不完善或者说有些问题
## 最近学习理解流程
1. new classpathxmlapplication
2. refersh
   1. ![](/技术学习流程/pic/2023-07-29-17-49-26.png)
3. obtainfreshbeanfactory
   1. ![](/技术学习流程/pic/2023-07-29-18-00-36.png)
   2. 通过这里面的loadbeandefination 将 <BEAN>的相关内容注入到容器中，此时没有进行初始化和实例化
   3. 主要通过recourceLoader读取配置文件、
   4. 将配置文件解析成beandefination
   5. 通过registerbean，将bean以bdf形式注入到map<name， beandefination>的map容器中
4. 像容器中注册beanpostprocess接口，用于初始化时候进行回调
   1. ![](/技术学习流程/pic/2023-07-29-18-06-08.png)
5. 初始化当前 ApplicationContext 的事件广播器，这里也不展开了 ---- initApplicationEventMulticaster
6. 开始初始化bean(初始化所有的 singleton beans,除了lazy-init标志的)
   1. ![](/技术学习流程/pic/2023-07-29-18-11-22.png)
   2. getbean - dogetbean - creatbean - docreatebean
   3. ![](/技术学习流程/pic/2023-07-29-18-20-33.png)实例化 - 属性注入 - 初始化
   4. initializeBean 这里面回调beanpostprocess 里的before和after的接口，在before和after中间去实现init-method 和  afterPropertiesSet （属于initiliazingBean）
   5. 在 beanpostprocess 的postProcessAfterInitialization里会去实现对bean的代理对象生成
      1. 这里涉及代理对象的生成，但是在aop中，出现循环依赖，代理对象需要提前暴露不能在最后暴露所以，在finshbeanfactoryinitialization里面中的createInstance之后设置属性之前调用
      2. ![](/技术学习流程/pic/2023-07-29-18-27-32.png)
      3. getEarlyBeanReference 提前暴露工厂方法到三级缓存中用户后续循环依赖的处理

### Autowired
![](/技术学习流程/pic/2023-07-29-18-40-25.png)
1. docreatebean中
2. 实例化完成，在applyMergedBeanDefinitionPostProcessors 去获取添加了Autowired属性的bean相关信息(元信息)
3. 在populateBean中对属性进行填充，通过反射的方式；执行**AutowiredAnnotationBeanPostProcessor**#postProcessProperties方法，进行相应注入
4. ![@Autowired的注入流程首先是需要构建一个InjectionMetadata，并通过InjectionMetadata的inject方法来进行注入](/技术学习流程/pic/2023-07-29-18-50-11.png)
5. 最终都会通过通过DefaultListableBeanFactory#resolveDependency获取依赖对象，然后通过反射进行相应属性赋值或者方法调用
6. 问题：Autowired注解默认根据Type查找依赖的bean，如果找到多个如何处理？
   1. 通过qualifier 指定名字， 或者 Primary（指定一个主要的），order好像也可以实现先用谁

### Resource
@Resource相当于@Autowired+@Qualifier，它可以直接指定bean的名称，而@Autowired不能直接指定，需要和@Qualifier配合使用
1. @Resource进行依赖查找的时候，首先是通过名称查找，如果匹配不到则退化到使用类型匹配；
2. @Autowired则是先通过类型查找，如果匹配到多个再通过名称查找
3. @Resource是通过**CommonAnnotationBeanPostProcessor**实现
   

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
