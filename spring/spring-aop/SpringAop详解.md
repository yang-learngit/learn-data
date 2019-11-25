# SpringAop详解

## 一、对AOP的初印象

**首先先给出一段比较专业的术语**（来自百度）

在软件业，AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方
式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

**然后我们举一个比较容易理解的例子**（来自：Spring 之 AOP）

要理解切面编程，就需要先理解什么是切面。用刀把一个西瓜分成两瓣，切开的切口就是切面；炒菜，锅与炉子共同来完成炒菜，锅与炉子就是切面。web层级设计中，web层->网关层->服务层->数据层，每一层之间也是一个切面。编程中，对象与对象之间，方法与方法之间，模块与模块之间都是一个个切面。

我们一般做活动的时候，一般对每一个接口都会做活动的有效性校验（是否开始、是否结束等等）、以及这个接口是不是需要用户登录。

按照正常的逻辑，我们可以这么做。

![](E:\我的\myGit\learn-data\spring\spring-aop\img\1-lizi.png)

这有个问题就是，有多少接口，就要多少次代码copy。对于一个“懒人”，这是不可容忍的。好，提出一个公共方法，每个接口都来调用这个接口。这里有点切面的味道了。

![](E:\我的\myGit\learn-data\spring\spring-aop\img\1-lizi2.png)

同样有个问题，我虽然不用每次都copy代码了，但是，每个接口总得要调用这个方法吧。于是就有了切面的概念，我将方法注入到接口调用的某个地方（切点）。

![](E:\我的\myGit\learn-data\spring\spring-aop\img\1-lizi3.png)

这样接口只需要关心具体的业务，而不需要关注其他非该接口关注的逻辑或处理。 
**红框处，就是面向切面编程。**



## 二、AOP的含义

AOP（Aspect-OrientedProgramming，面向方面编程），可以说是OOP（Object-Oriented Programing，面向对象编程）的补充和完善。OOP引入封装、继承和多态性等概念来建立一种对象层次结构，用以模拟公共行为的一个集合。当我们需要为分散的对象引入公共行为的时候，OOP则显得无能为力。也就是说，OOP允许你定义从上到下的关系，但并不适合定义从左到右的关系。例如日志功能。日志代码往往水平地散布在所有对象层次中，而与它所散布到的对象的核心功能毫无关系。对于其他类型的代码，如安全性、异常处理和透明的持续性也是如此。这种散布在各处的无关的代码被称为横切（cross-cutting）代码，在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。

而AOP技术则恰恰相反，它利用一种称为“横切”的技术，剖解开封装的对象内部，并将那些影响了多个类的公共行为封装到一个可重用模块，并将其名为"Aspect"，即方面。所谓“方面”，简单地说，就是将那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块间的耦合度，并有利于未来的可操作性和可维护性。AOP代表的是一个横向的关系，如果说“对象”是一个空心的圆柱体，其中封装的是对象的属性和行为；那么面向方面编程的方法，就仿佛一把利刃，将这些空心圆柱体剖开，以获得其内部的消息。而剖开的切面，也就是所谓的“方面”了。然后它又以巧夺天功的妙手将这些剖开的切面复原，不留痕迹。

使用“横切”技术，AOP把软件系统分为两个部分：核心关注点和横切关注点。业务处理的主要流程是核心关注点，与之关系不大的部分是横切关注点。横切关注点的一个特点是，他们经常发生在核心关注点的多处，而各处都基本相似。比如权限认证、日志、事务处理。Aop 的作用在于分离系统中的各种关注点，将核心关注点和横切关注点分离开来。正如Avanade公司的高级方案构架师Adam Magee所说，AOP的核心思想就是“将应用程序中的商业逻辑同对其提供支持的通用服务进行分离。”

实现AOP的技术，主要分为两大类：一是采用动态代理技术，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。

**三、AOP相关概念**

**1.连接点（Joinpoint）**

​      程序执行的某个特定位置：如类开始初始化之前、类初始化之后、类某个方法调用前、调用后等；一个类或一段程序代码拥有一些具有边界性质的特定点，这些代码中的特定点就成为“连接点”，Spring仅支持方法的连接点，即仅能在方法调用前、方法调用后以及方法调用前后的这些程序执行点织入增强。比如：黑客攻击系统需要找到突破口，没有突破口就没有办法攻击，从某种程度上来说，AOP就是一个黑客，连接点就是AOP向目标类攻击的候选点。

​      连接点有两个信息确定：第一是用方法表示的程序执行点；第二是用相对点表示的方位；如在Test.foo()方法执行前的连接点，执行点为Test.foo，方位为该方法执行前的位置。Spring使用切点对执行点进行定位，而方位则在增强类型中定义。

**2.切点（Pointcut）**

​      每个程序类都拥有许多连接点，如一个拥有两个方法的类，这两个方法都是连接点，即连接点是程序类中客观存在的事物。但在为数众多的连接点中，如何定位到某个连接点上呢？AOP通过切点定位特定连接点。通过数据库查询的概念来理解切点和连接点：连接点相当于数据库表中的记录，而切点相当于查询条件。连接点和切点不是一一对应的关系，一个切点可以匹配多个连接点。

​      在Spring中，切点通过org.springframework.aop.Pointcut接口进行描述，它使用类和方法作为连接点的查询条件，Spring AOP的规则解析引擎负责解析切点所设定的查询条件，找到对应的连接点；其实确切的说应该是执行点而非连接点，因为连接点是方法执行前、执行后等包括方位信息的具体程序执行点，而切点只定位到某个方法上，所以如果希望定位到具体连接点上，还需要提供方位信息。

**3.增强（Advice）**

​     增强是织入到目标类连接点上的一段程序代码（好比AOP以黑客的身份往业务类中装入木马），增强还拥有一个和连接点相关的信息，这便是执行点的方位。结合执行点方位信息和切点信息，我们就可以找到特定的连接点了，所以Spring提供的增强接口都是带方位名的：BefortAdvice、AfterReturningAdvice、ThrowsAdvice等。（有些将Advice翻译为通知，但通知就是把某个消息传达给被通知者，并没有为被通知者做任何事情，而Spring的Advice必须嵌入到某个类的连接点上，并完成了一段附加的应用逻辑；）

**4.目标对象（Target）**

​     增强逻辑的织入目标类，如果没有AOP，目标业务类需要自己实现所有逻辑，在AOP的帮助下，目标类只实现那些非横切逻辑的程序逻辑，而其他监测代码则可以使用AOP动态织入到特定的连接点上。

**5.引介（Introduction）**

​     引介是一种特殊的增强，它为类添加一些属性和方法，这样即使一个业务类原本没有实现某个接口，通过AOP的引介功能，我们可以动态的为该业务类添加接口的实现逻辑，让这个业务类成为这个接口的实现类。

**6.织入（Weaving）**

将 `Aspect` 和其他对象连接起来, 并创建 `Advice`d object 的过程。

织入是将增强添加到目标类具体连接点上的过程，AOP就像一台织布机，将目标类、增强或者引介编织到一起，AOP有三种织入的方式：
a.编译期间织入，这要求使用特殊的java编译器；
b.类装载期织入，这要求使用特殊的类装载器；
c.动态代理织入，在运行期为目标类添加增强生成子类的方式。
Spring采用动态代理织入，而AspectJ采用编译器织入和类装载期织入。

**7.代理（Proxy）**

一个类被AOP织入增强后，就产生出了一个结果类，它是融合了原类和增强逻辑的代理类。

**8.切面（Aspect）**

Aspect 声明类似于 Java 中的类声明，在 Aspect 中会包含着一些 Pointcut 以及相应的 Advice。

切面由切点和增强组成，它既包括了横切逻辑的定义，也包括了连接点的定义，Spring AOP就是负责实施切面的框架，它将切面所定义的横切逻辑织入到切面所指定的连接点中。



**然后举一个容易理解的例子：** 
看完了上面的理论部分知识, 我相信还是会有不少朋友感觉到 AOP 的概念还是很模糊, 对 AOP 中的各种概念理解的还不是很透彻. 其实这很正常, 因为 AOP 中的概念是在是太多了, 我当时也是花了老大劲才梳理清楚的. 
下面我以一个简单的例子来比喻一下 AOP 中 Aspect, Joint point, Pointcut 与 Advice之间的关系. 
让我们来假设一下, 从前有一个叫爪哇的小县城, 在一个月黑风高的晚上, 这个县城中发生了命案. 作案的凶手十分狡猾, 现场没有留下什么有价值的线索. 不过万幸的是, 刚从隔壁回来的老王恰好在这时候无意中发现了凶手行凶的过程, 但是由于天色已晚, 加上凶手蒙着面, 老王并没有看清凶手的面目, 只知道凶手是个男性, 身高约七尺五寸. 爪哇县的县令根据老王的描述, 对守门的士兵下命令说: 凡是发现有身高七尺五寸的男性, 都要抓过来审问. 士兵当然不敢违背县令的命令, 只好把进出城的所有符合条件的人都抓了起来.

来让我们看一下上面的一个小故事和 AOP 到底有什么对应关系. 
首先我们知道, 在 Spring AOP 中 Joint point 指代的是所有方法的执行点, 而 point cut 是一个描述信息, 它修饰的是 Joint point, 通过 point cut, 我们就可以确定哪些 Joint point 可以被织入 Advice. 对应到我们在上面举的例子, 我们可以做一个简单的类比, Joint point 就相当于 爪哇的小县城里的百姓,pointcut 就相当于 老王所做的指控, 即凶手是个男性, 身高约七尺五寸, 而 Advice 则是施加在符合老王所描述的嫌疑人的动作: 抓过来审问. 
为什么可以这样类比呢?

**Joint point** ： 爪哇的小县城里的百姓: 因为根据定义, Joint point 是所有可能被织入 Advice 的候选的点, 在 Spring AOP中, 则可以认为所有方法执行点都是 Joint point. 而在我们上面的例子中, 命案发生在小县城中, 按理说在此县城中的所有人都有可能是嫌疑人.

**Pointcut** ：男性, 身高约七尺五寸: 我们知道, 所有的方法(joint point) 都可以织入 Advice, 但是我们并不希望在所有方法上都织入 Advice, 而 Pointcut 的作用就是提供一组规则来匹配joinpoint, 给满足规则的 joinpoint 添加 Advice. 同理, 对于县令来说, 他再昏庸, 也知道不能把县城中的所有百姓都抓起来审问, 而是根据凶手是个男性, 身高约七尺五寸, 把符合条件的人抓起来. 在这里 凶手是个男性, 身高约七尺五寸 就是一个修饰谓语, 它限定了凶手的范围, 满足此修饰规则的百姓都是嫌疑人, 都需要抓起来审问.

**Advice** ：抓过来审问, Advice 是一个动作, 即一段 Java 代码, 这段 Java 代码是作用于 point cut 所限定的那些 Joint point 上的. 同理, 对比到我们的例子中, 抓过来审问 这个动作就是对作用于那些满足 男性, 身高约七尺五寸 的爪哇的小县城里的百姓.

**Aspect**：Aspect 是 point cut 与 Advice 的组合, 因此在这里我们就可以类比: “根据老王的线索, 凡是发现有身高七尺五寸的男性, 都要抓过来审问” 这一整个动作可以被认为是一个 Aspect.

**总结：AOP的工作重点就是如何将增强应用于目标对象的连接点上，这里首先包括两个工作：第一，如何通过切点和增强定位到连接点；第二，如何在增强中编写切面的代码。**

![](E:\我的\myGit\learn-data\spring\spring-aop\img\1-gainian.png)

## 三、其他的一些内容

AOP中的Joinpoint可以有多种类型：构造方法调用，字段的设置和获取，方法的调用，方法的执行，异常的处理执行，类的初始化。也就是说在AOP的概念中我们可以在上面的这些Joinpoint上织入我们自定义的Advice，但是在Spring中却没有实现上面所有的joinpoint，确切的说，Spring只支持方法执行类型的Joinpoint。

### Advice 的类型

**before advice**：在 join point 前被执行的 advice. 虽然 before advice 是在 join point 前被执行, 但是它并不能够阻止 join point 的执行, 除非发生了异常(即我们在 before advice 代码中, 不能人为地决定是否继续执行 join point 中的代码)

**after return advice**： 在一个 join point 正常返回后执行的 advice。

**after throwing advice** ：当一个 join point 抛出异常后执行的 advice。

**after(final) advice** ：无论一个 join point 是正常退出还是发生了异常, 都会被执行的 advice。

**around advice** ：在 join point 前和 joint point 退出后都执行的 advice. 这个是最常用的 advice。

**introduction**：introduction可以为原有的对象增加新的属性和方法。



## 四、AOP使用场景

AOP用来封装横切关注点，具体可以在下面的场景中使用:
Authentication 权限
Caching 缓存
Context passing 内容传递
Error handling 错误处理
Lazy loading　懒加载
Debugging　　调试
logging, tracing, profiling and monitoring　记录跟踪　优化　校准
Performance optimization　性能优化
Persistence　　持久化
Resource pooling　资源池
Synchronization　同步
Transactions 事务

## 五、实现原理

### 1、代理模式

**代理的概念：简单的理解就是通过为某一个对象创建一个代理对象，我们不直接引用原本的对象，而是由创建的代理对象来控制对原对象的引用。**

代理模式是常用的java设计模式，他的特征是代理类与委托类(或目标类)有同样的接口，代理类主要负责为委托类预处理消息、过滤消息、把消息转发给委托类，以及事后处理消息等。代理类与委托类之间通常会存在关联关系，一个代理类的对象与一个委托类的对象关联，代理类的对象本身并不真正实现服务，而是通过调用委托类的对象的相关方法，来提供特定的服务。 
按照代理的创建时期，代理类可以分为两种。 
**静态代理：**由程序员创建或特定工具自动生成源代码，再对其编译。在程序运行前，代理类的.class文件就已经存在了。 
**动态代理：**在程序运行时，运用反射机制动态创建而成，无需手动编写代码。动态代理不仅简化了编程工作，而且提高了软件系统的可扩展性，因为Java反射机制可以生成任意类型的动态代理类。

**代理原理：**代理对象内部含有对真实对象的引用，从而可以操作真实对象，同时代理对象提供与真实对象相同的接口以便在任何时刻都能代替真实对象。同时，代理对象可以在执行真实对象操作时，附加其他的操作，相当于对真实对象进行封装。

### 2、AOP的实现原理

**AOP的实现关键在于AOP框架自动创建的AOP代理。AOP代理主要分为两大类：**

**静态代理：使用AOP框架提供的命令进行编译，从而在编译阶段就可以生成AOP代理类，因此也称为编译时增强；静态代理一Aspectj为代表。**

**动态代理：在运行时借助于JDK动态代理，CGLIB等在内存中临时生成AOP动态代理类，因此也被称为运行时增强，Spring AOP用的就是动态代理。**

AOP分为静态AOP和动态AOP。

静态AOP是指AspectJ实现的AOP，他是将切面代码直接编译到Java类文件中。

动态AOP是指将切面代码进行动态织入实现的AOP。Spring的AOP为动态AOP，实现的技术为：**JDK提供的动态代理技术** 和 **CGLIB(动态字节码增强技术)**。**尽管实现技术不一样，但都是基于代理模式，都是生成一个代理对象。**

#### **（1）JDK动态代理**

**a. JDK动态代理是面向接口的，必须提供一个委托类和代理类都要实现的接口，只有接口中的方法才能够被代理。b. JDK动态代理的实现主要使用java.lang.reflect包里的Proxy类和InvocationHandler接口。**

**InvocationHandler接口：**

来看看java的API帮助文档是怎么样描述InvocationHandler接口的：

```java
InvocationHandler is the interface implemented by the invocation handler of a proxy instance. 

Each proxy instance has an associated invocation handler. When a method is invoked on a proxy instance, the method invocation is encoded and 
dispatched to the invoke method of its invocation handler.
```

说明：每一个动态代理类都必须要实现InvocationHandler这个接口，并且每个代理类的实例都关联到了一个handler，当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由InvocationHandler这个接口的 invoke 方法来进行调用。同时在invoke的方法里 我们可以对被代理对象的方法调用做增强处理(如添加事务、日志、权限验证等操作)。我们来看看InvocationHandler这个接口的唯一一个方法 invoke 方法：

```java
public interface InvocationHandler { 
     public Object invoke(Object proxy,Method method,Object[] args) throws Throwable; 
}
```

参数说明： 
Object proxy：指被代理的对象。 
Method method：要调用的方法。(指代的是我们所要调用代理对象的某个方法的Method对象) 
Object[] args：方法调用时所需要的参数。(指代的是调用真实对象某个方法时接受的参数)

可以将InvocationHandler接口的子类想象成一个代理的最终操作类，替换掉ProxySubject。 

**Proxy类：** 
Proxy类是专门完成代理的操作类，可以通过此类为一个或多个接口动态地生成实现类，此类提供了如下的操作方法：

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException 
```

参数说明： 
ClassLoader loader：类加载器 
Class<?>[] interfaces：得到全部的接口 
InvocationHandler h：得到InvocationHandler接口的子类实例

**JdkDynamicAopProxy源码：**

```java
package org.springframework.aop.framework;
 
import java.io.Serializable;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.List;
 
import org.aopalliance.intercept.MethodInvocation;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
 
import org.springframework.aop.AopInvocationException;
import org.springframework.aop.RawTargetAccess;
import org.springframework.aop.TargetSource;
import org.springframework.aop.support.AopUtils;
import org.springframework.util.Assert;
import org.springframework.util.ClassUtils;
 
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
 
	private static final long serialVersionUID = 5531744639992436476L;
 
	/*
         *注意：我们可以避免这个类和CGLIB之间的代码重复
        *代理通过重构“调用”到模板方法中。 但是，这种方法
        *与复制粘贴解决方案相比，至少增加了10％的性能开销，所以我们牺牲了
        *优雅的表现。 （我们有一个很好的测试套件来确保不同的
        *代理行为相同:-)
        *这样，我们也可以更轻松地利用每个班级中的次要优化。
        */
 
        / **我们使用静态日志来避免序列化问题* /
	private static final Log logger = LogFactory.getLog(JdkDynamicAopProxy.class);
 
	/** Config used to configure this proxy */
	private final AdvisedSupport advised;
 
      //代理接口上是否定义了{@link #hashCode}方法？
      private boolean equalsDefined;
    
        //代理接口上是否定义了{@link #hashCode}方法？private boolean hashCodeDefined; 
    /**为给定的AOP配置构造一个新的JdkDynamicAopProxy。
     * @param将AOP配置配置为AdvisedSupport对象
     *如果配置无效，则引发AopConfigException。 我们试图提供信息
     *在这种情况下是例外情况，而不是在稍后发生神秘故障。 
     */ 
    public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
        Assert.notNull(config, "AdvisedSupport must not be null");
        if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
            throw new AopConfigException("No advisors and no TargetSource specified");
        }
        this.advised = config;
    }
    @Overridepublic 
    Object getProxy() {
        return getProxy(ClassUtils.getDefaultClassLoader());
    }
    @Overridepublic 
    Object getProxy(ClassLoader classLoader) {
        if (logger.isDebugEnabled()) {
            logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
        }
        Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
        return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
    }
    private void findDefinedEqualsAndHashCodeMethods(Class<?>[] proxiedInterfaces) {
        for (Class<?> proxiedInterface : proxiedInterfaces) {
            Method[] methods = proxiedInterface.getDeclaredMethods();
            for (Method method : methods) {
                if (AopUtils.isEqualsMethod(method)) {
                    this.equalsDefined = true;
                }
                if (AopUtils.isHashCodeMethod(method)) {
                    this.hashCodeDefined = true;
                }
                if (this.equalsDefined && this.hashCodeDefined) {
                    return;
                }
            }
        }
    }
    /** 
    * Implementation of {@code InvocationHandler.invoke}. 
    * <p>Callers will see exactly the exception thrown by the target, 
    * unless a hook method throws an exception. 
    */
    @Overridepublic Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        MethodInvocation invocation;
        Object oldProxy = null;
        boolean setProxyContext = false;
        TargetSource targetSource = this.advised.targetSource;
        Class<?> targetClass = null;
        Object target = null;
        try {
            // eqauls()方法，具目标对象未实现此方法
            if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
                // The target does not implement the equals(Object) method itself.
                return equals(args[0]);
            }
            //hashCode()方法，具目标对象未实现此方法
            if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                // The target does not implement the hashCode() method itself.
                return hashCode();
            }
            //Advised接口或者其父接口中定义的方法,直接反射调用,不应用通知
            if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                // Service invocations on ProxyConfig with the proxy config...
                return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
            }
            Object retVal;
            if (this.advised.exposeProxy) {
                // Make invocation available if necessary.
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }
            // May be null. Get as late as possible to minimize the time we "own" the target,
            // in case it comes from a pool.
            //获得目标对象的类
            target = targetSource.getTarget();
            if (target != null) {
                targetClass = target.getClass();
            }
            // Get the interception chain for this method.
            // 获取可以应用到此方法上的Interceptor列表
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
            // Check whether we have any advice. If we don't, we can fallback on direct
            // reflective invocation of the target, and avoid creating a MethodInvocation.
            //如果没有可以应用到此方法的通知(Interceptor)，此直接反射调用 method.invoke(target, args)
            if (chain.isEmpty()) {
                // We can skip creating a MethodInvocation: just invoke the target directly
                // Note that the final invoker must be an InvokerInterceptor so we know it does
                // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
            }else {
                // We need to create a method invocation...
                invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                // Proceed to the joinpoint through the interceptor chain.
                //创建MethodInvocation
                retVal = invocation.proceed();
            }
            // Massage return value if necessary.
            Class<?> returnType = method.getReturnType();
            if (retVal != null && retVal == target && returnType.isInstance(proxy) &&!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                // Special case: it returned "this" and the return type of the method
                // is type-compatible. Note that we can't help if the target sets
                // a reference to itself in another returned object.
                retVal = proxy;}else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
            }
            return retVal;
        }finally {
            if (target != null && !targetSource.isStatic()) {
                // Must have come from TargetSource.
                targetSource.releaseTarget(target);
            }if (setProxyContext) {
                // Restore old proxy.
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }
    @Overridepublic 
    boolean equals(Object other) {
        if (other == this) {
            return true;
        }
        if (other == null) {
            return false;
        }
        JdkDynamicAopProxy otherProxy;
        if (other instanceof JdkDynamicAopProxy) {
            otherProxy = (JdkDynamicAopProxy) other;
        }else if (Proxy.isProxyClass(other.getClass())) {
            InvocationHandler ih = Proxy.getInvocationHandler(other);
            if (!(ih instanceof JdkDynamicAopProxy)) {
                return false;
            }
            otherProxy = (JdkDynamicAopProxy) ih;
        }else {
            // Not a valid comparison...
            return false;
        }
        // 如果我们到达这里，otherProxy是另一个AopProxy。
        return AopProxyUtils.equalsInProxy(this.advised, otherProxy.advised);
    }
    /** * 代理使用TargetSource的哈希码。. */
    @Overridepublic 
    int hashCode() {
        return JdkDynamicAopProxy.class.hashCode() * 13 + this.advised.getTargetSource().hashCode();
    }
}


```

**JDK动态代理示例：**

定义一个业务接口IUserService，如下：

```java
package com.spring.aop;

public interface IUserService {
    //添加用户
    public void addUser();
    //删除用户
    public void deleteUser();
}
```

一个简单的实现类UserServiceImpl，如下：

```java
package com.spring.aop;

public class UserServiceImpl implements IUserService{
    
    public void addUser(){
        System.out.println("新增了一个用户！");
    }
    
    public void deleteUser(){
        System.out.println("删除了一个用户！");
    }
}
```

现在我们要实现的是，在addUser和deleteUser之前和之后分别动态植入处理。
JDK动态代理主要用到java.lang.reflect包中的两个类：Proxy和InvocationHandler。
InvocationHandler是一个接口，通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态的将横切逻辑和业务逻辑编织在一起。
Proxy利用InvocationHandler动态创建一个符合某一接口的实例，生成目标类的代理对象。
如下，我们创建一个InvocationHandler实例DynamicProxy：**(当执行动态代理对象里的目标方法时，实际上会替换成调用DynamicProxy的invoke方法)**

```java
package com.spring.aop;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class DynamicProxy implements InvocationHandler{
    
    //被代理对象（就是要给这个目标类创建代理对象）
    private Object target;
    
    //传递代理目标的实例，因为代理处理器需要，也可以用set等方法。
    public DynamicProxy(Object target){
        this.target=target;
    }
    
    /**
     * 覆盖java.lang.reflect.InvocationHandler的方法invoke()进行织入(增强)的操作。
     * 这个方法是给代理对象调用的，留心的是内部的method调用的对象是目标对象，可别写错。
     * 参数说明：
     * proxy是生成的代理对象，method是代理的方法，args是方法接收的参数
     */
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
        //目标方法之前执行
        System.out.println("do sth Before...");
        //通过反射机制来调用目标类方法
        Object result = method.invoke(target, args);
        //目标方法之后执行
        System.out.println("do sth After...\n");
        return result;
    }
}
```

 下面是测试：

```java
package com.spring.aop;

//用java.lang.reflect.Proxy.newProxyInstance()方法创建动态实例来调用代理实例的方法
import java.lang.reflect.Proxy;

public class DynamicTest {
    
    public static void main(String[] args){
        //希望被代理的目标业务类
        IUserService target = new UserServiceImpl();
        //将目标类和横切类编织在一起
        DynamicProxy handler= new DynamicProxy(target);
        //创建代理实例，它可以看作是要代理的目标业务类的加多了横切代码(方法)的一个子类
        //创建代理实例(使用Proxy类和自定义的调用处理逻辑(handler)来生成一个代理对象)
        IUserService proxy = (IUserService)Proxy.newProxyInstance(
                target.getClass().getClassLoader(),//目标类的类加载器
                target.getClass().getInterfaces(), //目标类的接口
                handler); //横切类
        proxy.addUser();
        proxy.deleteUser();
    }
}
```

说明：上面的代码完成业务类代码和横切代码的编制工作，并生成了代理实例，newProxyInstance方法的第一个参数为类加载器，第二个参数为目标类所实现的一组接口，第三个参数是整合了业务逻辑和横切逻辑的编织器对象。

每一个动态代理实例的调用都要通过InvocationHandler接口的handler（调用处理器）来调用，动态代理不做任何执行操作，只是在创建动态代理时，把要实现的接口和handler关联，动态代理要帮助被代理执行的任务，要转交给handler来执行。其实就是调用invoke方法。(可以看到执行代理实例的addUser()和deleteUser()方法时执行的是DynamicProxy的invoke()方法。)

运行结果：

![](E:\我的\myGit\learn-data\spring\spring-aop\img\1-jdk动态代理例子.png)

基本流程：用Proxy类创建目标类的动态代理，创建时需要指定一个自己实现InvocationHandler接口的回调类的对象，这个回调类中有一个invoke()用于拦截对目标类各个方法的调用。创建好代理后就可以直接在代理上调用目标对象的各个方法。

实现动态代理步骤：
A. 创建一个实现接口InvocationHandler的类，他必须实现invoke方法。
B．创建被代理的类以及接口。
C．通过Proxy的静态方法newProxyInstance（ClassLoader loader, Class<?>[]interfaces, InvocationHandler handler）创建一个代理。
D．通过代理调用方法。

**使用JDK动态代理有一个很大的限制，就是它要求目标类必须实现了对应方法的接口，它只能为接口创建代理实例。**我们在上文测试类中的Proxy的newProxyInstance方法中可以看到，该方法第二个参数便是目标类的接口。如果该类没有实现接口，这就要靠cglib动态代理了。

#### **（2）CGLIB动态代理**

CGLib采用非常底层的字节码技术，可以为一个类创建一个子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，并顺势植入横切逻辑。

字节码生成技术实现AOP，其实就是继承被代理对象，然后Override需要被代理的方法，在覆盖该方法时，自然是可以插入我们自己的代码的。因为需要Override被代理对象的方法，所以自然用CGLIB技术实现AOP时，就必须要求需要被代理的方法不能是final方法，因为final方法不能被子类覆盖。

a.使用CGLIB动态代理不要求必须有接口，生成的代理对象是目标对象的子类对象，**所以需要代理的方法不能是private或者final或者static的。**
b.使用CGLIB动态代理需要有对cglib的jar包依赖（导入asm.jar和cglib-nodep-2.1_3.jar）

CGLibProxy与JDKProxy的代理机制基本类似，只是其动态代理的代理对象并非某个接口的实现，而是针对目标类扩展的子类。换句话说JDKProxy返回动态代理类，是目标类所实现接口的另一个实现版本，它实现了对目标类的代理（如同UserDAOProxy与UserDAOImp的关系），而CGLibProxy返回的动态代理类，则是目标代理类的一个子类（代理类扩展了UserDaoImpl类）

**cglib 代理特点：**
CGLIB 是针对类来实现代理，它的原理是对指定的目标类生成一个子类，并覆盖其中方法。因为采用的是继承，**所以不能对 finall 类进行继承**。

我们使用CGLIB实现上面的例子：

代理的最终操作类：

```java
package com.spring.aop;

import java.lang.reflect.Method;

import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

public class CglibProxy implements MethodInterceptor{
    
    //增强器，动态代码生成器
    Enhancer enhancer = new Enhancer();
    
    /**
     * 创建代理对象
     * @param clazz
     * @return 返回代理对象
     */
    public Object getProxy(Class clazz){
        //设置父类，也就是被代理的类(目标类)
        enhancer.setSuperclass(clazz);
        //设置回调（在调用父类方法时，回调this.intercept()）
        enhancer.setCallback(this);
        //通过字节码技术动态创建子类实例(动态扩展了UserServiceImpl类)
        return enhancer.create();
    }
    
    /**
     * 拦截方法：在代理实例上拦截并处理目标方法的调用，返回结果
     * obj:目标对象代理的实例;
     * method:目标对象调用父类方法的method实例;
     * args:调用父类方法传递参数;
     * proxy:代理的方法去调用目标方法
     */
    public Object intercept(Object obj,Method method,Object[] args,MethodProxy proxy) 
        throws Throwable{
        
        System.out.println("--------测试intercept方法的四个参数的含义-----------");
        System.out.println("obj:"+obj.getClass());
        System.out.println("method:"+method.getName());
        System.out.println("proxy:"+proxy.getSuperName());
        if(args!=null&&args.length>0){
            for(Object value : args){
                System.out.println("args:"+value);
            }
        }

        //目标方法之前执行
        System.out.println("do sth Before...");
        //目标方法调用
        //通过代理类实例调用父类的方法，即是目标业务类方法的调用
        Object result = proxy.invokeSuper(obj, args);
        //目标方法之后执行
        System.out.println("do sth After...\n");
        return result;
    }
}
```

测试类：

```java
package com.spring.aop;

public class CglibProxyTest {
    
    public static void main(String[] args){
        CglibProxy proxy=new CglibProxy();
        //通过java.lang.reflect.Proxy的getProxy()动态生成目标业务类的子类，即是代理类，再由此得到代理实例
        //通过动态生成子类的方式创建代理类
        IUserService target=(IUserService)proxy.getProxy(UserServiceImpl.class);
        target.addUser();
        target.deleteUser();
    }
}
```

基本流程：需要自己写代理类，它实现MethodInterceptor接口，有一个intercept()回调方法用于拦截对目标方法的调用，里面使用methodProxy来调用目标方法。创建代理对象要用Enhance类，用它设置好代理的目标类、由intercept()回调的代理类实例、最后用create()创建并返回代理实例。

输出：

![](E:\我的\myGit\learn-data\spring\spring-aop\img\1-cglib动态代理例子.png)

我们看到达到了同样的效果。它的原理是生成一个父类enhancer.setSuperclass(clazz)的子类enhancer.create()，然后对父类的方法进行拦截enhancer.setCallback(this). 对父类的方法进行覆盖，所以父类方法不能是final的。

**总结：** 
　　(1).通过输出可以看出，最终调用的是com.spring.aop.UserServiceImpl的子类(也是代理类)com.spring.aop.UserServiceImpl$$EnhancerByCGLIB$$43831205的方法。
　　(2). private,final和static修饰的方法不能被代理。

**注意：**
　　(1).CGLIB是通过实现目标类的子类来实现代理，不需要定义接口。
　　(2).生成代理对象使用最多的是通过Enhancer和继承了Callback接口的MethodInterceptor接口来生成代理对象，设置callback对象的作用是当调用代理对象方法的时候会交给callback对象的来处理。
　　(3).创建子类对象是通过使用Enhancer类的对象，通过设置enhancer.setSuperClass(Class class)和enhancer.setCallback(Callback callback)来创建代理对象。

解释MethodInterceptor接口的intercept方法：

```java
Object intercept(Object var1, Method var2, Object[] var3, MethodProxy var4) throws Throwable;
```

参数说明：Object var1代表的是子类代理对象，Method var2代表的是要调用的方法反射对象，第三个参数是传递给调用方法的参数，前三个参数和JDK的InvocationHandler接口的invoke方法中参数含义是一样的，第四个参数MethodProxy对象是cglib生成的用来代替method对象的，使用此对象会比jdk的method对象的效率要高。

如果使用method对象来调用目标对象的方法: method.invoke(var1, var3)，则会陷入无限递归循环中， 因为此时的目标对象是目标类的子代理类对象。

MethodProxy类提供了两个invoke方法：

```java
public Object invokeSuper(Object obj, Object[] args) throws Throwable;
public Object invoke(Object obj, Object[] args) throws Throwable;
```

注意此时应该使用invokeSuper()方法，顾名思义调用的是父类的方法，若使用invoke方法，则需要提供一个目标类对象，但我们只有目标类子类代理对象，所以会陷入无限递归循环中。

CGLIB所创建的动态代理对象的性能比JDK所创建的动态代理对象的性能高很多，但创建动态代理对象时比JDK创建动态代理对象要花费更长的时间。

#### （3）总结

**JDK代理和CGLIB代理的总结(生成代理对象的前提是有AOP切入)**

 (1)、如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP。 如果就是单纯的用IOC生成一个对象，也没有AOP的切入不会生成代理的，只会NEW一个实例，给Spring的Bean工厂。
(2)、如果目标对象实现了接口，可以强制使用CGLIB实现AOP 
如何强制使用CGLIB实现AOP 
\* 添加CGLIB库 
\* 在spring配置文件中加入<aop:aspectj-autoproxy proxy-target-class="true"/>就能强制使用 
(3)、如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换（没有实现接口的就用CGLIB代理，使用了接口的类就用JDK动态代理）

#### （4）区别

**JDK动态代理和CGLIB字节码生成的区别：**
(1)、JDK动态代理只能对实现了接口的类生成代理，而不能针对类。CGLIB是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法。因为是继承，所以该类或方法最好不要声明成final。

(2)、JDK代理是不需要依赖第三方的库，只要JDK环境就可以进行代理，它有几个要求
\* 实现InvocationHandler;
\* 使用Proxy.newProxyInstance产生代理对象;
\* 被代理的对象必须要实现接口;
CGLib 必须依赖于CGLib的类库，但是它需要类来实现任何接口代理的是指定的类生成一个子类，覆盖其中的方法，是一种继承。

(3)、jdk的核心是实现InvocationHandler接口，使用invoke()方法进行面向切面的处理，调用相应的通知。cglib的核心是实现MethodInterceptor接口，使用intercept()方法进行面向切面的处理，调用相应的通知。

### 3、小结

​	AOP 广泛应用于处理一些具有横切性质的系统级服务，AOP 的出现是对 OOP 的良好补充，它使得开发者能用更优雅的方式处理具有横切性质的服务。不管是哪种 AOP 实现，不论是 AspectJ、还是 Spring AOP，它们都需要动态地生成一个 AOP 代理类，区别只是生成 AOP 代理类的时机不同：

AspectJ 采用编译时生成 AOP 代理类，因此具有更好的性能，但需要使用特定的编译器进行处理；而 Spring AOP 则采用运行时生成 AOP 代理类，因此无需使用特定编译器进行处理。由于 Spring AOP 需要在每次运行时生成 AOP 代理，因此性能略差一些。

### 4、Spring AOP代理对象的生成

Spring提供了两种方式来生成代理对象: JDKProxy和Cglib，具体使用哪种方式生成由AopProxyFactory根据AdvisedSupport对象的配置来决定。默认的策略是如果目标类是接口，则使用JDK动态代理技术，否则使用Cglib来生成代理。下面我们来研究一下Spring如何使用JDK来生成代理对象，具体的生成代码放在JdkDynamicAopProxy这个类中，直接上相关代码：

```java
/**
    * <ol>
    * <li>获取代理类要实现的接口,除了Advised对象中配置的,还会加上SpringProxy, Advised(opaque=false)
    * <li>检查上面得到的接口中有没有定义 equals或者hashcode的接口
    * <li>调用Proxy.newProxyInstance创建代理对象
    * </ol>
    */
   public Object getProxy(ClassLoader classLoader) {
       if (logger.isDebugEnabled()) {
           logger.debug("Creating JDK dynamic proxy: target source is " +this.advised.getTargetSource());
       }
       Class[] proxiedInterfaces =AopProxyUtils.completeProxiedInterfaces(this.advised);
       findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
       return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}

```

代理对象生成了，那切面是如何织入的？

我们知道InvocationHandler是JDK动态代理的核心，生成的代理对象的方法调用都会委托到InvocationHandler.invoke()方法。而通过JdkDynamicAopProxy的签名我们可以看到这个类其实也实现了InvocationHandler，下面我们就通过分析这个类中实现的invoke()方法来具体看下Spring AOP是如何织入切面的。

```java
publicObject invoke(Object proxy, Method method, Object[] args) throwsThrowable {
       MethodInvocation invocation = null;
       Object oldProxy = null;
       boolean setProxyContext = false;
 
       TargetSource targetSource = this.advised.targetSource;
       Class targetClass = null;
       Object target = null;
 
       try {
           //eqauls()方法，具目标对象未实现此方法
           if (!this.equalsDefined && AopUtils.isEqualsMethod(method)){
                return (equals(args[0])? Boolean.TRUE : Boolean.FALSE);
           }
 
           //hashCode()方法，具目标对象未实现此方法
           if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)){
                return newInteger(hashCode());
           }
 
           //Advised接口或者其父接口中定义的方法,直接反射调用,不应用通知
           if (!this.advised.opaque &&method.getDeclaringClass().isInterface()
                    &&method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                // Service invocations onProxyConfig with the proxy config...
                return AopUtils.invokeJoinpointUsingReflection(this.advised,method, args);
           }
 
           Object retVal = null;
 
           if (this.advised.exposeProxy) {
                // Make invocation available ifnecessary.
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
           }
 
           //获得目标对象的类
           target = targetSource.getTarget();
           if (target != null) {
                targetClass = target.getClass();
           }
 
           //获取可以应用到此方法上的Interceptor列表
           List chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method,targetClass);
 
           //如果没有可以应用到此方法的通知(Interceptor)，此直接反射调用 method.invoke(target, args)
           if (chain.isEmpty()) {
                retVal = AopUtils.invokeJoinpointUsingReflection(target,method, args);
           } else {
                //创建MethodInvocation
                invocation = newReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                retVal = invocation.proceed();
           }
 
           // Massage return value if necessary.
           if (retVal != null && retVal == target &&method.getReturnType().isInstance(proxy)
                    &&!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                // Special case: it returned"this" and the return type of the method
                // is type-compatible. Notethat we can't help if the target sets
                // a reference to itself inanother returned object.
                retVal = proxy;
           }
           return retVal;
       } finally {
           if (target != null && !targetSource.isStatic()) {
                // Must have come fromTargetSource.
               targetSource.releaseTarget(target);
           }
           if (setProxyContext) {
                // Restore old proxy.
                AopContext.setCurrentProxy(oldProxy);
           }
       }
    }

```

主流程可以简述为：获取可以应用到此方法上的通知链（Interceptor Chain）,如果有,则应用通知,并执行joinpoint; 如果没有,则直接反射执行joinpoint。而这里的关键是通知链是如何获取的以及它又是如何执行的，下面逐一分析下。

首先，从上面的代码可以看到，通知链是通过Advised.getInterceptorsAndDynamicInterceptionAdvice()这个方法来获取的,我们来看下这个方法的实现:

```java
public List<Object>getInterceptorsAndDynamicInterceptionAdvice(Method method, Class targetClass) {
                   MethodCacheKeycacheKey = new MethodCacheKey(method);
                   List<Object>cached = this.methodCache.get(cacheKey);
                   if(cached == null) {
                            cached= this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                                               this,method, targetClass);
                            this.methodCache.put(cacheKey,cached);
                   }
                   returncached;
         }

```

可以看到实际的获取工作其实是由AdvisorChainFactory. getInterceptorsAndDynamicInterceptionAdvice()这个方法来完成的，获取到的结果会被缓存。

下面来分析下这个方法的实现：

```java

/**
    * 从提供的配置实例config中获取advisor列表,遍历处理这些advisor.如果是IntroductionAdvisor,
    * 则判断此Advisor能否应用到目标类targetClass上.如果是PointcutAdvisor,则判断
    * 此Advisor能否应用到目标方法method上.将满足条件的Advisor通过AdvisorAdaptor转化成Interceptor列表返回.
    */
    publicList getInterceptorsAndDynamicInterceptionAdvice(Advised config, Methodmethod, Class targetClass) {
       // This is somewhat tricky... we have to process introductions first,
       // but we need to preserve order in the ultimate list.
       List interceptorList = new ArrayList(config.getAdvisors().length);
 
       //查看是否包含IntroductionAdvisor
       boolean hasIntroductions = hasMatchingIntroductions(config,targetClass);
 
       //这里实际上注册一系列AdvisorAdapter,用于将Advisor转化成MethodInterceptor
       AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
 
       Advisor[] advisors = config.getAdvisors();
        for (int i = 0; i <advisors.length; i++) {
           Advisor advisor = advisors[i];
           if (advisor instanceof PointcutAdvisor) {
                // Add it conditionally.
                PointcutAdvisor pointcutAdvisor= (PointcutAdvisor) advisor;
                if(config.isPreFiltered() ||pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {
                    //TODO: 这个地方这两个方法的位置可以互换下
                    //将Advisor转化成Interceptor
                    MethodInterceptor[]interceptors = registry.getInterceptors(advisor);
 
                    //检查当前advisor的pointcut是否可以匹配当前方法
                    MethodMatcher mm =pointcutAdvisor.getPointcut().getMethodMatcher();
 
                    if (MethodMatchers.matches(mm,method, targetClass, hasIntroductions)) {
                        if(mm.isRuntime()) {
                            // Creating a newobject instance in the getInterceptors() method
                            // isn't a problemas we normally cache created chains.
                            for (intj = 0; j < interceptors.length; j++) {
                               interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptors[j],mm));
                            }
                        } else {
                            interceptorList.addAll(Arrays.asList(interceptors));
                        }
                    }
                }
           } else if (advisor instanceof IntroductionAdvisor){
                IntroductionAdvisor ia =(IntroductionAdvisor) advisor;
                if(config.isPreFiltered() || ia.getClassFilter().matches(targetClass)) {
                    Interceptor[] interceptors= registry.getInterceptors(advisor);
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
           } else {
                Interceptor[] interceptors =registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
           }
       }
       return interceptorList;
}
```

这个方法执行完成后，Advised中配置能够应用到连接点或者目标类的Advisor全部被转化成了MethodInterceptor.

接下来我们再看下得到的拦截器链是怎么起作用的。

```java
if (chain.isEmpty()) {
                retVal = AopUtils.invokeJoinpointUsingReflection(target,method, args);
            } else {
                //创建MethodInvocation
                invocation = newReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                retVal = invocation.proceed();
            }
```

从这段代码可以看出，如果得到的拦截器链为空，则直接反射调用目标方法，否则创建MethodInvocation，调用其proceed方法，触发拦截器链的执行，来看下具体代码：

```java
public Object proceed() throws Throwable {
       //  We start with an index of -1and increment early.
       if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size()- 1) {
           //如果Interceptor执行完了，则执行joinPoint
           return invokeJoinpoint();
       }
 
       Object interceptorOrInterceptionAdvice =
           this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
       
       //如果要动态匹配joinPoint
       if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher){
           // Evaluate dynamic method matcher here: static part will already have
           // been evaluated and found to match.
           InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
           //动态匹配：运行时参数是否满足匹配条件
           if (dm.methodMatcher.matches(this.method, this.targetClass,this.arguments)) {
                //执行当前Intercetpor
                returndm.interceptor.invoke(this);
           }
           else {
                //动态匹配失败时,略过当前Intercetpor,调用下一个Interceptor
                return proceed();
           }
       }
       else {
           // It's an interceptor, so we just invoke it: The pointcutwill have
           // been evaluated statically before this object was constructed.
           //执行当前Intercetpor
           return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
       }
}

```

