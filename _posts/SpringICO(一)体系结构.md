#### 总览
![](https://ws1.sinaimg.cn/large/006tNc79gy1fzayqbge1xj31100k0jta.jpg)

##### Resource 模块
Resource，对资源的抽象，它的每一个实现类都代表了一种资源的访问策略，如ClasspathResource 、 URLResource ，FileSystemResource 等。
![](https://ws1.sinaimg.cn/large/006tNc79gy1g04u2tdynaj31fe0h0wga.jpg)
有了资源，就应该有资源加载，Spring 利用 ResourceLoader 来进行统一资源加载，类图如下：
![](https://ws1.sinaimg.cn/large/006tNc79gy1g04u39umahj318k0d4wgo.jpg)
#### BeanFactory 模块
BeanFactory 是一个非常纯粹的 bean 容器，它是 IOC 必备的数据结构，其中 BeanDefinition 是她的基本结构，它内部维护着一个 BeanDefinition map ，并可根据 BeanDefinition 的描述进行 bean 的创建和管理。
![](https://ws1.sinaimg.cn/large/006tNc79gy1g04u4j582pj317w0m076o.jpg)
BeanFacoty 有三个直接子类 `ListableBeanFactory`、`HierarchicalBeanFactory` 和 `AutowireCapableBeanFactory`、`DefaultListableBeanFactory` 为最终默认实现，它实现了所有接口。
#### Beandefinition 模块
BeanDefinition 用来描述 Spring 中的 Bean 对象。
![](https://ws4.sinaimg.cn/large/006tNc79gy1g04u6j27q0j30mi0h0jsg.jpg)
#### BeandefinitionReader模块
BeanDefinitionReader 的作用是读取 Spring 的配置文件的内容，并将其转换成 Ioc 容器内部的数据结构：BeanDefinition。
![](https://ws2.sinaimg.cn/large/006tNc79gy1g04u79gv7xj315q0cmjsx.jpg)
#### ApplicationContext体系
这个就是大名鼎鼎的 `Spring 容器`，它叫做应用上下文，与我们应用息息相关，她继承 BeanFactory，所以它是 BeanFactory 的扩展升级版，如果BeanFactory 是屌丝的话，那么 ApplicationContext 则是名副其实的高富帅。由于 ApplicationContext 的结构就决定了它与 BeanFactory 的不同，其主要区别有：
>继承 MessageSource，提供国际化的标准访问策略。

    BeanFactory是不支持国际化功能的，因为BeanFactory没有扩展Spring中MessageResource接口。相反，由于ApplicationContext扩展了MessageResource接口，因而具有消息处理的能力(i18N)，具体spring如何使用国际化，
   
   
>继承 ApplicationEventPublisher ，提供强大的事件机制。

>扩展 ResourceLoader，可以用来加载多个 Resource，可以灵活访问不同的资源。

>对 Web 应用的支持。

    与BeanFactory通常以编程的方式被创建不同，ApplicationContext能以声明的方式创建，如使用ContextLoader。当然你也可以使用ApplicationContext的实现之一来以编程的方式创建ApplicationContext实例 。  




