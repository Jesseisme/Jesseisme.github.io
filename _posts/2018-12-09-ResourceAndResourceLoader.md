---
title: "SpringIOC 加载资源"
subtitle: "Spring源码学习"
date:       2018-12-09 23:13:20
author: "Jesse"
tags:
  - Spring
---
[TOC]

#### 资源的定义 

org.springframework.core.io.Resource 是 Spring 框架所有资源的抽象和访问接口，继承自 org.springframework.core.io.InputStreamSource接口。作为所有资源的统一抽象，Source 定义了一些通用的方法。其子类AbstractResource提供了默认的实现定义如下： 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0stxz66k9j30660413yh.jpg) 

```java 

public interface Resource extends InputStreamSource { 

boolean exists();//资源是否存在 

default boolean isReadable() { 

return true; 

} 

//资源所代表的句柄是否被一个stream打开了 

default boolean isOpen() { 

return false; 

} 

default boolean isFile() { 

return false; 

} 

URL getURL() throws IOException; 

URI getURI() throws IOException; 

File getFile() throws IOException; 

default ReadableByteChannel readableChannel() throws IOException { 

return Channels.newChannel(getInputStream()); 

} 

long contentLength() throws IOException; 

long lastModified() throws IOException; 

//根据资源的相对路径创建新资源 

Resource createRelative(String relativePath) throws IOException; 

@Nullable 

String getFilename(); 

//资源的描述 

String getDescription(); 

} 

```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1g0stypjfdyj31fe0h0wga.jpg) 

resource体系结构图 

---

图中可以看到最下面的一排实现类，对不同资源的类型，提供不同的策略。 

>FileSystemResource:只要是跟 File 打交道的 

>ByteArrayResource:字节数组 

>UrlResource：对 java.net.URL类型资源的封装 

>ClassPathResource：class path 类型资源的实现。使用给定的 ClassLoader 或者给定的 Class 来加载资源。 

>InputStreamResource 

如果要实现自定义的 Resource，记住不要实现 Resource 接口，而应该继承 AbstractResource 抽象类，然后根据当前的具体资源特性覆盖相应的方法。 

#### 资源的加载 

resource是资源的抽象定义。resourceLoader是对资源的加载。 

org.springframework.core.io.ResourceLoader 为 Spring 资源加载的统一抽象，具体的实现由其子类实现。 

```java 

public interface ResourceLoader { 

String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX; 

Resource getResource(String location); 

ClassLoader getClassLoader(); 

} 

```

![](https://ws4.sinaimg.cn/large/006tKfTcly1g0y1c4mteij319e0ek0tr.jpg)
##### DefaultResourceLoader
DefaultResourceLoader是resourceLoader的默认实现。
ResourceLoader 中最核心的方法为`getResource()`,它根据提供的 location 返回相应的 Resource，而 DefaultResourceLoader 对该方法提供了核心实现(它的两个子类都没有提供覆盖该方法，所以可以断定ResourceLoader 的资源加载策略就封装 DefaultResourceLoader中)，如下：

```java
   public Resource getResource(String location) {
        Assert.notNull(location, "Location must not be null");

        for (ProtocolResolver protocolResolver : this.protocolResolvers) {
            Resource resource = protocolResolver.resolve(location, this);
            if (resource != null) {
                return resource;
            }
        }

        if (location.startsWith("/")) {
            return getResourceByPath(location);
        }
        else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
            return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
        }
        else {
            try {
                // Try to parse the location as a URL...
                URL url = new URL(location);
                return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
            }
            catch (MalformedURLException ex) {
                // No URL -> resolve as resource path.
                return getResourceByPath(location);
            }
        }
    }
```

首先通过 ProtocolResolver 来加载资源，成功返回 Resource，否则调用如下逻辑：

* 若 location 以 / 开头，则调用 `getResourceByPath()`构造 ClassPathContextResource 类型资源并返回。

* 若 location 以 classpath: 开头，则构造 ClassPathResource 类型资源并返回，在构造该资源时，通过 `getClassLoader()`获取当前的 ClassLoader。

* 其他则构造 URL ，尝试通过它进行资源定位，若没有抛出 MalformedURLException 异常，则判断是否为 FileURL , 如果是则构造 FileUrlResource 类型资源，否则构造 UrlResource。若在加载过程中抛出 MalformedURLException 异常，则委派 `getResourceByPath()` 实现资源定位加载。 

开头遍历的ProtocolResolver 是用户自定义协议资源解决策略，作为 DefaultResourceLoader 的 SPI，它允许用户自定义资源加载协议，而不需要继承 ResourceLoader 的子类。在介绍 Resource 时，提到如果要实现自定义 Resource，我们只需要继承 DefaultResource 即可，但是有了 ProtocolResolver 后，我们不需要直接继承 DefaultResourceLoader，改为实现 ProtocolResolver 接口也可以实现自定义的 ResourceLoader。 ProtocolResolver 接口，仅有一个方法`Resource resolve(String location, ResourceLoader resourceLoader)`，该方法接收两个参数：资源路径location，指定的加载器 ResourceLoader，返回为相应的 Resource 。在 Spring 中你会发现该接口并没有实现类，它需要用户自定义，自定义的 Resolver 如何加入 Spring 体系呢？调用 `DefaultResourceLoader.addProtocolResolver()` 即可，如下：

```java
    public void addProtocolResolver(ProtocolResolver resolver) {
        Assert.notNull(resolver, "ProtocolResolver must not be null");
        this.protocolResolvers.add(resolver);
    }
```

