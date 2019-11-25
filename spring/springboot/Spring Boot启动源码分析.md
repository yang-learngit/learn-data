# Spring Boot启动源码分析

下面我们来看源码，走进启动模式；

```java
SpringApplication.run(BootApplication.class, args);
```

启动类开始执行；

```java
new SpringApplication(primarySources).run(args);

public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}
```


先走进SpringApplication的构造方法，new一个SpringApplication实例，然后执行run方法。run方法执行结束之后整个启动流程就是走完了。

1、SpringApplication 的构造函数，创建SpringApplication对象，一下是构造函数实现的具体初始化过程。

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	//使用的资源加载器
    	this.resourceLoader = resourceLoader;
    //主要的bean资源 primarySources【在这里是启动类所在的.class】,不能为null,如果为null，抛异常
		Assert.notNull(primarySources, "PrimarySources must not be null");
    //primarySources 是Set<Class<?>> 类型，把启动类（我这里是定义为BootApplication）
    //.class (Class<?> 该类的实例数组转化成list，放在LinkedHashSet集合中)
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    /**
      * deduceWebApplicationType() = SERVER
      * 应用程序应该作为基于servlet的web应用程序运行，并且应该启动一个
      * 嵌入式servlet web服务器。
      */
		this.webApplicationType = deduceWebApplicationType();
    /**
      * 使用类加载机制，和发射实现对ApplicationContextInitializer 接口熟悉的初始化。
      * ApplicationContextInitializer是Spring IOC一个回调接口，它会在ConfigurableApplicationContext的refresh()
      * 方法调用之前被调用,做一些容器的初始化工作。每一个initailizer都是一个实现了ApplicationContextInitializer接口的实例。
      * springboot中一共有6个实现类，后面遇到再分析各个实现类初始化的作用。
      */
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
    /**  
      * 使用类加载机制，和发射实现对ApplicationListener(事件监听容器) 接口熟悉的初始化。
      * 当用Spring ApplicationContext注册时，事件将相应地过滤，并调用侦听器以匹配事件对象。
      */
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 指定main函数启动所在的类，即启动类BootApplication.class
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

 

2、启动运行Spring应用程序，创建并刷新一个新应用application程序

```java
public ConfigurableApplicationContext run(String... args) {
    /**
        StopWatch: 简单的秒表，允许定时的一些任务，公开每个指定任务的总运行时间和运行时间。
        这个对象的设计不是线程安全的，没有使用同步。不会作为生产环境中使用，二此处因为启动
        SpringApplication是在单线程环境下，使用安全。
    */
    StopWatch stopWatch = new StopWatch();
    // 设置当前启动的时间为系统时间startTimeMillis = System.currentTimeMillis();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    /** 系统设置headless模式，
    （是一种系统配置，其中显示设备、键盘或鼠标缺乏。听起来出乎意料，但实际上您可以在这种模式下执行不同的操作，即使是使用图形数据。）
     想进一步了解的访问（https://www.oracle.com/technetwork/articles/javase/headless-136834.html）
    */
    configureHeadlessProperty();
    /** 
    通过类加载机制和反射创建SpringApplicationRunListeners实例SpringApplicationRunListeners:在创建和更新ApplicationContext 方法前后分别触发回调的形式调用了listeners对象的started方法和finished....等等方法, 并在创建和刷新ApplicationContext的不同阶段，调用listeners的相应方法以执行操作。所以，SpringApplicationRunListeners实际上就是在SpringApplication 对象的run方法执行的不同阶段，去监听回调一些操作，并且这些操作是可配置的。  通过spring.factories中发现SpringApplicationRunListeners 加载的是org.springframework.boot.context.event.EventPublishingRunListener是 SpringApplicationRunListener的实现类.(SpringApplicationRunListeners的内部存储了SpringApplicationRunListener的集合，提供了SpringApplicationRunListener一样方  法，方便统一遍历调用所有SpringApplicationRunListener。)
        */
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 开始监听指定的一些event事件,查看源码springboot中应该是有45处会触发回调
    listeners.starting();
    try {
        // 提供对用于运行SpringApplication的参数的访问。取默认实现
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
        // 构建容器环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                                                                 applicationArguments);
        // 对环境中一些bean忽略配置
        configureIgnoreBeanInfo(environment);
        // 日志控制台打印设置
        Banner printedBanner = printBanner(environment);
        // 创建容器
        context = createApplicationContext();
        exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
        /**
        追踪源码prepareContext（）进去我们可以发现容器准备阶段做了下面的事情：
        容器设置配置环境，并且监听容器，初始化容器，记录启动日志，
        将给定的singleton对象添加到此工厂的singleton缓存中。
        将bean加载到应用程序上下文中。
        */
        prepareContext(context, environment, listeners, applicationArguments,
                       printedBanner);
        /**
        1、同步刷新，对上下文的bean工厂包括子类的刷新准备使用，初始化此上下文的消息源，注册拦截bean的处理器，检查侦听器bean并注册它们，实例化所有剩余的(非延迟-init)单例。
        2、异步开启一个同步线程去时时监控容器是否被关闭，当关闭此应用程序上下文，销毁其bean工厂中的所有bean。 如果我们注册了一个jvm shutdown hook，那就不需要单独去开启一个线程去监控了
        */
        refreshContext(context);
        // 刷新容器之后无任何操作，里面是空的方法
        afterRefresh(context, applicationArguments);
        // stopwatch 的作用就是记录启动消耗的时间，和开始启动的时间等信息记录下来
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass)
                .logStarted(getApplicationLog(), stopWatch);
        }
        /** 
        启动监听容器中所有的事件:
        	上下文已经刷新，应用程序已经启动，但是没有调用CommandLineRunners和applicationrunner。
        */
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    /**
    启动监听:
    	当应用程序上下文被刷新，所有CommandLineRunners和applicationrunner都被调用时，在运行方法完成之前立即调用。
    */
    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```




上面的源码中多处都是内部是用了下面源码中的几个方法，其实本质就是解析加载spring-boot-2.0.3.RELEASE.jar/META-INF/spring.factories
中的配置，来创建初始化我们需要的Application context/listeners/Loaders。

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
	return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
	SpringApplicationRunListener.class, types, this, args));
}
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
	return getSpringFactoriesInstances(type, new Class<?>[] {});
}
 
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, Object... args) {
	ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
	// Use names and ensure unique to protect against duplicates
	Set<String> names = new LinkedHashSet<>(
			SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
			classLoader, args, names);
	AnnotationAwareOrderComparator.sort(instances);
	return instances;
}
```


