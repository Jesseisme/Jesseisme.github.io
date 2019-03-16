---
title: "SpringAOP源码浅析"
subtitle: "读Spring源码"
date:       2018-01-22 23:34:20
author: "Jesse"
tags:
  - spring
---
[TOC]

### 前言

​	IOC初始化的时候会把一个个的Bean解析成BeanDefinition，并把它放到容器中的ConcurrentHashMap里面。然后当我们需要某个bean的时候，可以通过getBean的方法获取实例(大部分是依赖注入不需要手动调用)。getBean方法里面使用了BeanPostProcessor这个接口

```java
public interface BeanPostProcessor {
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
```

这两个方法分别在bean初始化之前和之后之前执行，这是Spring为了便于扩展预留的方法。AOP的实现就是在这两个方法里面实现的。

​	Spring AOP 的实现原理——**动态代理**。

代理模式：接口 + 真实实现类 + 代理类，其中 真实实现类 和 代理类 都要实现接口，实例化的时候要使用代理类。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1g077o9kn58j30d4069glu.jpg)

AOP做的就是通过JDK Proxy或CGLIB 动态生成这个代理类，然后利用这个代理类实现对外的服务。

### 简单的例子

两个接口

```java
// OrderService.java
public interface OrderService {

    Order createOrder(String username, String product);

    Order queryOrder(String username);
}
// UserService.java
public interface UserService {

    User createUser(String firstName, String lastName, int age);

    User queryUser();
}
```

实现类

```java
// OrderServiceImpl.java
public class OrderServiceImpl implements OrderService {

    @Override
    public Order createOrder(String username, String product) {
        Order order = new Order();
        order.setUsername(username);
        order.setProduct(product);
        return order;
    }

    @Override
    public Order queryOrder(String username) {
        Order order = new Order();
        order.setUsername("test");
        order.setProduct("test");
        return order;
    }
}

// UserServiceImpl.java
public class UserServiceImpl implements UserService {

    @Override
    public User createUser(String firstName, String lastName, int age) {
        User user = new User();
        user.setFirstName(firstName);
        user.setLastName(lastName);
        user.setAge(age);
        return user;
    }

    @Override
    public User queryUser() {
        User user = new User();
        user.setFirstName("test");
        user.setLastName("test");
        user.setAge(20);
        return user;
    }
}
```

两个advice

```java
public class LogArgsAdvice implements MethodBeforeAdvice {

    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("准备执行方法: " + method.getName() + ", 参数列表：" + Arrays.toString(args));
    }
}
```

```java
public class LogResultAdvice implements AfterReturningAdvice {

    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target)
            throws Throwable {
        System.out.println(method.getName() + "方法返回：" + returnValue);
    }
}
```

配置文件

```xml
 <bean id="userServiceImpl" class="cn.jessej.springaoplearning.service.imple.UserServiceImpl"/>
    <bean id="orderServiceImpl" class="cn.jessej.springaoplearning.service.imple.OrderServiceImpl"/>

    <!--定义两个 advice-->
    <bean id="logArgsAdvice" class="cn.jessej.springaoplearning.aop_spring_1_2.LogArgsAdvice"/>
    <bean id="logResultAdvice" class="cn.jessej.springaoplearning.aop_spring_1_2.LogResultAdvice"/>

    <!--定义两个 advisor-->
    <!--记录 create* 方法的传参-->
    <bean id="logArgsAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice" ref="logArgsAdvice" />
        <property name="pattern" value="cn.jessej.springaoplearning.service.*.create.*" />
    </bean>
    <!--记录 query* 的返回值-->
    <bean id="logResultAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice" ref="logResultAdvice" />
        <property name="pattern" value="cn.jessej.springaoplearning.service.*.query.*" />
    </bean>

    <!--定义DefaultAdvisorAutoProxyCreator-->
    <!--DefaultAdvisorAutoProxyCreator 使得所有的 advisor 配置自动生效-->
    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />

```

启动类

```java
public class SpringAopSourceApplication {

   public static void main(String[] args) {

      // 启动 Spring 的 IOC 容器
      ApplicationContext context = new ClassPathXmlApplicationContext("classpath:DefaultAdvisorAutoProxy.xml");

      UserService userService = context.getBean(UserService.class);
      OrderService orderService = context.getBean(OrderService.class);

      userService.createUser("Jesse", "Cruise", 55);
      userService.queryUser();

      orderService.createOrder("SuSan", "随便买点什么");
      orderService.queryOrder("Suan");
   }
}
```

### IOC 容器管理 AOP 实例

DefaultAdvisorAutoProxyCreator实现了BeanPostProcessor，Aware，ProxyConfig。

![](https://ws3.sinaimg.cn/large/006tKfTcly1g14xxyos3ij30h8067wet.jpg)

看一下bean加载时的源码：

```java
#AbstractAutowireCapableBeanFactory#
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
            throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 1. 创建实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    ...

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 2. 装载属性
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 3. 初始化
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    ...
}
```

initializeBean(...) 方法中会调用 BeanPostProcessor 中的方法，如下：

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
   ...
   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      // 1. 执行每一个 BeanPostProcessor 的 postProcessBeforeInitialization 方法
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      // 调用 bean 配置中的 init-method="xxx"
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   ...
   if (mbd == null || !mbd.isSynthetic()) {
      // 我们关注的重点是这里！！！
      // 2. 执行每一个 BeanPostProcessor 的 postProcessAfterInitialization 方法
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }
   return wrappedBean;
}

```

DefaultAdvisorAutoProxyCreator 中的postProcessAfterInitialization() 方法在其父类 AbstractAutoProxyCreator 中被实现了。

```java
#AbstractAutoProxyCreator#
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // 返回匹配当前 bean 的所有的 advisor、advice、interceptor!!!!!!!!!!!!!!!!!!
   // 对于本文的例子，"userServiceImpl" 和 "OrderServiceImpl" 这两个 bean 创建过程中，
   //   到这边的时候都会返回两个 advisor
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      // 创建代理!!!!!!!!!!!!!!!!!!!!!!!!!!!!
     //TargetSource是用于封装真正的实现类
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

createProxy(…) 方法：

```java
protected Object createProxy(
      Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

   // 创建 ProxyFactory 实例
   ProxyFactory proxyFactory = new ProxyFactory();
   proxyFactory.copyFrom(this);

   // 在 schema-based 的配置方式中，我们介绍过，如果希望使用 CGLIB 来代理接口，可以配置
   // proxy-target-class="true",这样不管有没有接口，都使用 CGLIB 来生成代理：
   //   <aop:config proxy-target-class="true">......</aop:config>
   if (!proxyFactory.isProxyTargetClass()) {
      if (shouldProxyTargetClass(beanClass, beanName)) {
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         // 点进去稍微看一下代码就知道了，主要就两句：
         // 1. 有接口的，调用一次或多次：proxyFactory.addInterface(ifc);
         // 2. 没有接口的，调用：proxyFactory.setProxyTargetClass(true);
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }

   // 这个方法会返回匹配了当前 bean 的 advisors 数组
   // 对于本文的例子，"userServiceImpl" 和 "OrderServiceImpl" 到这边的时候都会返回两个 advisor
   // 注意：如果 specificInterceptors 中有 advice 和 interceptor，它们也会被包装成 advisor，进去看下源码就清楚了
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   for (Advisor advisor : advisors) {
      proxyFactory.addAdvisor(advisor);
   }

   proxyFactory.setTargetSource(targetSource);
   customizeProxyFactory(proxyFactory);

   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }

   return proxyFactory.getProxy(getProxyClassLoader());
}
```

剩下的都在getProxy()方法里面了

### ProxyFactory 详解

先看getProxy()

```java
#ProxyFactory#
public Object getProxy(ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}
```
#### createAopProxy()方法
```java
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this);
	}

```
##### getAopProxyFactory()
```java
#ProxyCreatorSupport#
	public AopProxyFactory getAopProxyFactory() {
		return this.aopProxyFactory;
	}
```
这是在ProxyCreatorSupport类下面，
这个类的构造器如下：
```java
	/**
	 * Create a new ProxyCreatorSupport instance.
	 */
	public ProxyCreatorSupport() {
		this.aopProxyFactory = new DefaultAopProxyFactory();
	}

```
这边得到了一个DefaultAopProxyFactory然后看下
它的 createAopProxy(…) 方法：
```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

   @Override
   public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
      // (默认false) || (proxy-target-class=true) || (没有接口)
      if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
         Class<?> targetClass = config.getTargetClass();
         if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                  "Either an interface or a target is required for proxy creation.");
         }
         // 如果要代理的类本身就是接口，也会用 JDK 动态代理
         if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
         }
         return new ObjenesisCglibAopProxy(config);
      }
      else {
         // 如果有接口，会跑到这个分支
         return new JdkDynamicAopProxy(config);
      }
   }
   // 判断是否有实现自定义的接口
   private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
      Class<?>[] ifcs = config.getProxiedInterfaces();
      return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
   }

}
```
所以这个方法的功能就是判断是返回JdkDynamicAopProxy还是ObjenesisCglibAopProxy

#### 两种AopProxy的getProxy(classLoader)
回到
```Java
public Object getProxy(ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}
```

看下AopProxy 实现类的 getProxy(classLoader) 实现。
##### JdkDynamicAopProxy的getProxy
![](https://ws2.sinaimg.cn/large/006tKfTcly1g14z4dmfraj30c803st8t.jpg)
JdkDynamicAopProxy实现了AopProxy和InvicationHandler
```java
@Override
public Object getProxy(ClassLoader classLoader) {
   if (logger.isDebugEnabled()) {
      logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
   }
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```
java.lang.reflect.Proxy.newProxyInstance(…) 方法需要三个参数
>1.ClassLoader
>2.需要实现的接口
>3.InvocationHandler
因为JdkDynamicAopProxy实现了InvocationHandler所以这边写的this

###### JdkDynamicAopProxy的invoke()
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   MethodInvocation invocation;
   Object oldProxy = null;
   boolean setProxyContext = false;

   TargetSource targetSource = this.advised.targetSource;
   Class<?> targetClass = null;
   Object target = null;

   try {
      if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
         // The target does not implement the equals(Object) method itself.
         // 代理的 equals 方法
         return equals(args[0]);
      }
      else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
         // The target does not implement the hashCode() method itself.
         // 代理的 hashCode 方法
         return hashCode();
      }
      else if (method.getDeclaringClass() == DecoratingProxy.class) {
         // There is only getDecoratedClass() declared -> dispatch to proxy config.
         // 
         return AopProxyUtils.ultimateTargetClass(this.advised);
      }
      else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
            method.getDeclaringClass().isAssignableFrom(Advised.class)) {
         // Service invocations on ProxyConfig with the proxy config...
         return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
      }

      Object retVal;

      // 如果设置了 exposeProxy，那么将 proxy 放到 ThreadLocal 中
      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy);
         setProxyContext = true;
      }
      // May be null. Get as late as possible to minimize the time we "own" the target,
      // in case it comes from a pool.
      target = targetSource.getTarget();
      if (target != null) {
         targetClass = target.getClass();
      }

      // Get the interception chain for this method.
      // 创建一个 chain，包含所有要执行的 advice
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      if (chain.isEmpty()) {
         // We can skip creating a MethodInvocation: just invoke the target directly
         // Note that the final invoker must be an InvokerInterceptor so we know it does
         // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
         // chain 是空的，说明不需要被增强，这种情况很简单
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {
         // We need to create a method invocation...
         // 执行方法，得到返回值
         invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         // Proceed to the joinpoint through the interceptor chain.
         retVal = invocation.proceed();
      }

      // Massage return value if necessary.
      Class<?> returnType = method.getReturnType();
      if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
         // Special case: it returned "this" and the return type of the method
         // is type-compatible. Note that we can't help if the target sets
         // a reference to itself in another returned object.
         retVal = proxy;
      }
      else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
         throw new AopInvocationException(
               "Null return value from advice does not match primitive return type for: " + method);
      }
      return retVal;
    }
   finally {
      if (target != null && !targetSource.isStatic()) {
         // Must have come from TargetSource.
         targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```
##### ObjenesisCglibAopProxy的getProxy(classLoader)。
![](https://ws1.sinaimg.cn/large/006tKfTcly1g14zd78hg0j30ca05imxe.jpg)

### 基于注解的 Spring AOP
AnnotationAwareAspectJAutoProxyCreator
![](https://ws2.sinaimg.cn/large/006tKfTcly1g14zvk3ij7j30gs0ewgmk.jpg)
AnnotationAwareAspectJAutoProxyCreator也是一个BeanPostProcessor。