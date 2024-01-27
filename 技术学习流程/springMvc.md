#  springMvc
## WebApplicationContext容器的初始化
### Root WebApplicationContext 容器
![](/技术学习流程/pic/2023-11-25-21-44-56.png)
root WebApplicationContext 通过ContextLoaderListener 监听到 Servlet 容器启动事件，则调用父类 ContextLoader 的 initWebApplicationContext(ServletContext servletContext) 方法，初始化 root WebApplicationContext 容器
默认使用：org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.**XmlWebApplicationContext**

### Servlet WebApplicationContext 容器
除了会初始化一个 Root WebApplicationContext 容器外，注入一个 DispatcherServlet 对象，初始化该对象的过程也会初始化一个 Servlet WebApplicationContext 容器
![](/技术学习流程/pic/2023-11-25-21-52-26.png)
HttpServletBean ： 负责将 ServletConfig 设置到当前 Servlet 对象中
FrameworkServlet： 负责初始化 Servlet WebApplicationContext 容器；该类覆写了 doGet、doPost 等方法，并将所有类型的请求委托给 doService 方法去处理，doService 是一个抽象方法，需要子类实现
DispatcherServlet： 负责初始化 Spring MVC 的各个组件，以及处理客户端的请求，协调各个组件工作
#### HttpServletBean
解析 <init-param/> 标签，封装到 PropertyValues pvs 中
然后配置到servlet中
initServletBean（）交由子类实现就是FrameworkServlet

#### FrameworkServlet
属性：
1. contextClass ： 创建的 WebApplicationContext 类型，默认为 XmlWebApplicationContext.class，在 Root WebApplicationContext 容器的创建过程中也是它
2. contextConfigLocation： 配置文件的地址，例如：classpath:spring-mvc.xml
3. webApplicationContext ：WebApplicationContext 对象
   1. 通过 findWebApplicationContext() 方法
   2. 通过 createWebApplicationContext(WebApplicationContext parent)
   3. 实现了 ApplicationContextAware 接口
   4. 通过上面的构造方法

FrameworkServlet.initServletBean:当前主要是初始化Servlet WebApplicationContext 容器
1. 调用 **initWebApplicationContext**() 方法，初始化 Servlet WebApplicationContext 对象
1. 获得 WebApplicationContext wac 对象，有三种情况
   1. 如果构造方法已经传入 webApplicationContext 属性，则直接引用给
   2. 调用 findWebApplicationContext()方法，从 ServletContext 获取对应的 WebApplicationContext 对象
   3. 调用createWebApplicationContext(@Nullable WebApplicationContext parent)方法，创建一个 
   4. WebApplicationContext 对象
2. 创建完成：则调用 onRefresh(ApplicationContext context) 方法，主动触发刷新事件
   1. 交给DispatcherServlet实现
   2. onRefresh有两种方式发生了解下：
      1. initWebApplicationContext 方法中直接调用
      2. 通过在 configureAndRefreshWebApplicationContext 方法中，触发 wac 的刷新事件
         1. configureAndRefreshWebApplicationContext添加监听器：这个是WebApplicationContext创建时候的第一种方式也就是通过构造方法的方式创建的
   
#### DispatcherServlet

##### DispatcherServlet流程图
![](/技术学习流程/pic/2023-11-27-17-42-49.png)

DispatcherServlet.onRefresh() -> initStrategies(context):初始化9大组建
```java 
protected void initStrategies(ApplicationContext context) {
    // 初始化 MultipartResolver
    initMultipartResolver(context);
    // 初始化 LocaleResolver
    initLocaleResolver(context);
    // 初始化 ThemeResolver
    initThemeResolver(context);
    // 初始化 HandlerMappings
    initHandlerMappings(context);
    // 初始化 HandlerAdapters
    initHandlerAdapters(context);
    // 初始化 HandlerExceptionResolvers 
    initHandlerExceptionResolvers(context);
    // 初始化 RequestToViewNameTranslator
    initRequestToViewNameTranslator(context);
    // 初始化 ViewResolvers
    initViewResolvers(context);
    // 初始化 FlashMapManager
    initFlashMapManager(context);
}
```
#### 总结：
Root WebApplicationContext 和 Servlet WebApplicationContext 容器，它们是父子关系
前者是在 Tomcat 或者 Jetty 等 Servlet 容器启动后，由 ContextLoaderListener 监听到相应事件而创建的，
后者是在 DispatcherServlet 初始化的过程中创建的，因为它是一个 HttpServlet 对象，会调用其 init 方法，完成初始化相关工作


## 9大组建
![](/技术学习流程/pic/2023-11-25-22-20-51.png)
1. 请求怎么来的：
   1. HttpServlet 的：doGet / doPost /doDelete .....
   2. FrameworkServlet 覆写上述方法
   3. FrameworkServlet 的 service(HttpServletRequest request, HttpServletResponse response) 方法，用于处理请求
   4. 最终调用方法：processRequest() ->doService()
   5. ![](/技术学习流程/pic/2023-11-25-22-24-46.png)
   6. **doService - 〉 doDispatch**（开始进入DispatcherServlet）

## doDispatch
![](/技术学习流程/pic/2023-11-27-17-52-33.png)
![](/技术学习流程/pic/2023-11-27-17-54-35.png)
![](/技术学习流程/pic/2023-11-27-17-55-16.png)
1. 检测请求是否为上传请求，如果是则通过 multipartResolver 将其封装成 MultipartHttpServletRequest 对象
   1. 使用checkMultipart - >通过multipartResolver -> 变成MultipartHttpServletRequest
2. 获得请求对应的 HandlerExecutionChain 对象（HandlerMethod 和 HandlerInterceptor 拦截器们
   1. getHandler
3. 获得当前 handler 对应的 HandlerAdapter 对象
   1. getHandlerAdapter
4. 前置处理 拦截器(方法如果有一个拦截器的前置处理返回false，则开始倒序触发所有的拦截器的 已完成处理)
5. 真正的调用 handler 方法，也就是执行对应的方法，并返回视图
6. 无视图的情况下设置默认视图名称
   1. applyDefaultViewName
7. 后置处理 拦截器
8. 记录异常(此处仅仅记录，不会抛出异常，而是统一交给 <9> 处理)
9.  处理正常和异常的请求调用结果
    1.  processDispatchResult
    2.  如果 ModelAndView 不为空且没有被清理，例如你现在使用最多的 @ResponseBody不需要渲染
10. 已完成处理 拦截器
11. 如果是上传请求则清理资源



#### MultipartResolver
主要解析文件上传的请求。例如，MultipartResolver 会将 HttpServletRequest 封装成 MultipartHttpServletRequest 对象，便于获取参数信息以及上传的文件

在 Spring Boot 中，multipartResolver 默认为 **StandardServletMultipartResolver** 实现类；基于Servlet 3.0 标准的上传文件 API 的 MultipartResolver 实现类

通过StandardServletMultipartResolver 解析HttpServletRequest 变成 StandardMultipartHttpServletRequest
![](/技术学习流程/pic/2023-11-27-18-08-23.png)
![](/技术学习流程/pic/2023-11-27-18-08-49.png)
总结：
1. MultipartResolver 对请求的 Content-Type 为 multipart/* 处理
2. 通过 MultipartResolver 组件对请求进行转换处理
   1. MultipartResolver两种实现：StandardServletMultipartResolver（springboot的默认实现）
      1. 由 Servlet 3.0 提供 API 获取请求中的 javax.servlet.http.Part 对象，然后进行解析，文件会封装成 StandardMultipartFile 对象
   2. CommonsMultipartResolver
      1. 基于 Apache Commons FileUpload 的 MultipartResolver 实现类
      2. 通过 Commons FileUpload 组件实现，将文件封装
      3. 成CommonsMultipartFile
3. 封装成 MultipartHttpServletRequest 对象便于获取参数信息和操作上传的文件即（MultipartFile 对象）

#### AbstractHandlerMapping
**HandlerMapping 组件，请求的处理器匹配器，负责为请求找到合适的 HandlerExecutionChain 处理器执行链，包含处理器（handler）和拦截器们（interceptors）**
**handler**：处理器是Object类型，可以将其理解成HandlerMethod对象（例如我们使用最多的@RequestMapping注解所标注的方法会解析成该对象），包含了方法的所有信息，通过该对象能够执行该方法
**HandlerInterceptor**：拦截器对处理请求进行增强处理，可用于在执行方法前、成功执行方法后、处理完成后进行一些逻辑处理
![](/技术学习流程/pic/2023-12-03-17-38-02.png)
通过HandlerMapping 获取链路是有顺序的，获取奥chain后就不会后面的遍历 
BeanNameUrlHandlerMapping -> RequestMappingHandlerMapping
![](/技术学习流程/pic/2023-12-03-17-43-34.png)
红色框用于解析：@RequestMapping
黄色框通过配置文件配置url路径![](/技术学习流程/pic/2023-12-03-17-47-06.png)
initHandlerMappings： 两个默认实现类![](/技术学习流程/pic/2023-12-03-18-14-22.png)
**gettHandler**
getHandler(HttpServletRequest request) 方法，获得请求对应的 HandlerExecutionChain 处理器执行链，包含处理器（handler）和拦截器们（interceptors）
![](/技术学习流程/pic/2023-12-03-19-56-25.png)
总结：AbstractHandlerMapping 抽象类，作为一个基类，实现了“为请求找到合适的 HandlerExecutionChain 处理器执行链”对应的的骨架逻辑，而暴露 **getHandlerInternal(HttpServletRequest request)** 抽象方法，交由子类实现。
这个获取handler后面痛过getHandlerExecutionChain 获取处理器和过滤器：HandlerExecutionChain

#### HandlerInterceptor
HandlerInterceptor 拦截器对处理请求进行增强处理，可用于在执行方法前、成功执行方法后、处理完成后进行一些逻辑处理
![](/技术学习流程/pic/2023-12-03-20-03-31.png)
springboot中
![](/技术学习流程/pic/2023-12-03-20-18-22.png)
![](/技术学习流程/pic/2023-12-03-20-20-10.png)
![](/技术学习流程/pic/2023-12-03-20-21-35.png)

#### AbstractHandlerMethodMapping
该系是基于 Method 进行匹配。例如，我们所熟知的 @RequestMapping 等注解的方式。一共就三个类，不多😈😈😈（红色框）
![](/技术学习流程/pic/2023-12-03-20-23-25.png)
![](/技术学习流程/pic/2023-12-03-20-32-19.png)
createHandlerMethod：创建
![](/技术学习流程/pic/2023-12-03-20-43-47.png)
![](/技术学习流程/pic/2023-12-03-20-46-19.png)

#### AbstractUrlHandlerMapping
![](/技术学习流程/pic/2023-12-03-20-51-23.png)
registerHandler：
   registerHandler(String[] urlPaths, String beanName) 方法，注册多个 URL 的处理器，方法如下：

![](/技术学习流程/pic/2023-12-03-20-55-30.png)
![](/技术学习流程/pic/2023-12-03-20-57-16.png)

##### SimpleUrlHandlerMapping
![](/技术学习流程/pic/2023-12-03-21-00-46.png)
##### BeanNameUrlHandlerMapping
![](/技术学习流程/pic/2023-12-03-21-01-14.png)
![](/技术学习流程/pic/2023-12-03-21-05-51.png)