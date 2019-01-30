# JAVA SPI机制（service provider interface服务提供者接口）

## 简介

SPI 全称为 (Service Provider Interface) ，是JDK内置的一种服务提供发现机制。

一个服务(Service)通常指的是已知的接口或者抽象类，服务提供方就是对这个接口或者抽象类的实现，然后按照SPI 标准存放到资源路径META-INF/services目录下，文件的命名为该服务接口的全限定名。

在java.util.ServiceLoader的文档里有比较详细的介绍。简单的总结下java spi机制的思想。我们系统里抽象的各个模块，往往有很多不同的实现方案，比如日志模块的方案，xml解析模块、jdbc模块的方案等。面向的对象的设计里，我们一般推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。为了实现在模块装配的时候能不在程序里动态指明，这就需要一种服务发现机制。 java spi就是提供这样的一个机制：为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要。

## SPI具体约定

- 在META-INF/services/目录中创建以Service接口全限定名命名的文件，该文件内容为Service接口具体实现类的全限定名，文件编码必须为UTF-8。

- 使用ServiceLoader.load(Class class); 动态加载Service接口的实现类。

- 如SPI的实现类为jar，则需要将其放在当前程序的classpath下。

- Service的具体实现类必须有一个不带参数的构造方法。

 java spi的具体约定为:当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。

jdk提供服务实现查找的一个工具类：java.util.ServiceLoader

```java
//将服务声明的文件名称定义为: example.spi.service.IService，与接口名称一致，其中的内容包括：
//example.spi.service.PrintServiceImpl  
//example.spi.service.EchoServiceImpl  

public static void main(String[] args) {  
    //实例化具体类时需要注意对应类有无参构造函数
    ServiceLoader<Service> serviceLoader = ServiceLoader.load(IService.class);  
   
    for (IService service : serviceLoader) {  
        service.printInfo();  
    }  
}
```

### ServiceLoader的源码分析

重要属性：

```java
// 加载的接口
private Class<S> service;

// 用于缓存已经加载的接口实现类，其中key为实现类的完整类名
private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

// 用于延迟加载接口的实现类
private LazyIterator lookupIterator;
```

第一步：获取一个ServiceLoader<Service> serviceLoader = ServiceLoader.load(IService.class);实例，此时还没有进行任何接口实现类的加载操作，属于延迟加载类型的。只是创建了LazyIterator lookupIterator对象而已。

第二步：ServiceLoader实现了Iterable接口，即实现了该接口的iterator()方法，实现内容如下：

```
// for循环遍历ServiceLoader的过程其实就是调用上述hasNext()和next()方法的过程
    public Iterator<S> iterator() {
        return new Iterator<S>() {

            Iterator<Map.Entry<String,S>> knownProviders
                = providers.entrySet().iterator();

            public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
                // 第一次循环遍历会使用lookupIterator去查找，之后就缓存到providers中。
　　　　　　　　　 // LazyIterator会去加载类路径下/META-INF/services/接口全称 文件的url地址，文件加载并解析完成之后，得到一系列的接口实现类的完整类名。
　　　　　　　　　 // 调用next()方法时才回去真正执行接口实现类的加载操作，并根据无参构造器创建出一个实例，存到providers中。
                return lookupIterator.hasNext();
            }

            public S next() {
                //再次遍历ServiceLoader，就直接遍历providers中的数据
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

        };
    }    
```

### ServiceLoader缺点

1. 虽然ServiceLoader使用延迟加载，但是基本只能通过遍历全部获取，也就是接口的实现类全部加载并实例化一遍。如果你并不想用某些实现类，它也被加载并实例化了，这就造成了浪费。
2. 获取某个实现类的方式不够灵活，只能通过Iterator形式获取，不能根据某个参数来获取对应的实现类。





## 应用场景

### common-logging  

apache最早提供的日志的门面接口。只有接口，没有实现。具体方案由各提供商实现， 发现日志提供商是通过扫描 META-INF/services/org.apache.commons.logging.LogFactory配置文件，通过读取该文件的内容找到日志提工商实现类。只要我们的日志实现里包含了这个文件，并在文件里制定 LogFactory工厂接口的实现类即可。   

###  JDBC

 jdbc4.0以前， 开发人员还需要基于Class.forName("xxx")的方式来装载驱动，jdbc4也基于spi的机制来发现驱动提供商了，可以通过META-INF/services/java.sql.Driver文件里指定实现类的方式来暴露驱动提供者.

### Dubbo SPI

 dubbo的扩展机制与JAVA SPI比较类似，但额外增加了其他功能：

1. 可以根据接口名称来获取服务，dubbo spi 可以通过getExtension(String key)的方法方便的获取某一个想要的扩展实现，java的SPI机制需要加载全部的实现类。

2. 服务声明文件支持A=B的方式，此时A为名称B为实现类。 文件名:com.alibaba.dubbo.rpc.Filter

   - echo=com.alibaba.dubbo.rpc.filter.EchoFilter
   - generic=com.alibaba.dubbo.rpc.filter.GenericFilter
   - genericimpl=com.alibaba.dubbo.rpc.filter.GenericImplFilter

3. 支持扩展IOC依赖注入功能，可以为Service之间的依赖关系注入相关的服务并保证单例。

   - 举例来说：接口A，实现者A1、A2。接口B，实现者B1、B2。

     现在实现者A1含有setB()方法，会自动注入一个接口B的实现者，此时注入B1还是B2呢？

     都不是，而是注入一个动态生成的接口B的实现者B$Adpative，该实现能够根据参数的不同，自动引用B1或者B2来完成相应的功能。

     Protocol$Adpative是根据URL参数中protocol属性的值来选择具体的实现类的。

     如值为dubbo，则从ExtensionLoader<Protocol>中获取dubbo对应的实例，即DubboProtocol实例

     如值为hessian，则从ExtensionLoader<Protocol>中获取hessian对应的实例，即HessianProtocol实例

     也就是说Protocol$Adpative能够根据url中的protocol属性值动态的采用对应的实现。

4. 对扩展采用装饰器模式进行功能增强，类似AOP实现的功能

   - 接口A的另一个实现者AWrapper1。在获取某一个接口A的实现者A1的时候，已经自动被AWrapper1包装了。

     ```java
     private A a;
     AWrapper1（A a）{
         this.a=a;
     }
     ```



## ExtensionLoader源码分析 

```
ExtensionLoader<Protocol> protocolLoader = ExtensionLoader.getExtensionLoader(Protocol.class);
Protocol protocol = protocolLoader.getAdaptiveExtension();

@Extension("dubbo")
public interface Protocol {

    int getDefaultPort();
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
    void destroy();
}
```

第一步：根据要加载的接口创建出一个ExtensionLoader实例

重要属性: ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();

用于缓存所有的扩展加载实例，这里加载Protocol.class，就以Protocol.class为key，创建的ExtensionLoader为value存储到上述EXTENSION_LOADERS中。

第二步:ExtensionLoader实例是加载Protocol的实现类

1. 先解析Protocol上的Extension注解的name,存至String cachedDefaultName属性中，作为默认的实现
2. 到类路径下的加载 META-INF/services/com.alibaba.dubbo.rpc.Protocol文件，然后就是读取每一行内容，加载对应的class。(扩展配置文件: /META-INF/dubbo/internal，/META-INF/dubbo/，META-INF/services)
3. 上述class分成三种情况来处理,对于一个接口的实现者，ExtensionLoader分三种情况来分别存储对应的实现者，属性分别如下：Class<?> cachedAdaptiveClass；Set<Class<?>> cachedWrapperClasses；Reference<Map<String, Class<?>>> cachedClasses；
   - 情况1： 如果这个class含有Adaptive注解，则将这个class设置为Class<?> cachedAdaptiveClass。
   - 情况2： 尝试获取有对应接口参数的构造器，如果能够获取到，则说明这个class是一个装饰类即需要存到Set<Class<?>> cachedWrapperClasses中
   - 情况3： 如果没有上述构造器。则获取class上的Extension注解，根据该注解的定义的name作为key，存至Reference<Map<String, Class<?>>> cachedClasses结构中 。

 

