# Spring中的@AliasFor标签

> 在Spring的众多注解中，经常会发现很多注解的不同属性起着相同的作用，比如@RequestMapping的value属性和path属性，这就需要做一些基本的限制，比如value和path的值不能冲突，比如任意设置value或者设置path属性的值，都能够通过另一个属性来获取值等等。为了统一处理这些情况，Spring创建了@AliasFor标签。

## 使用

@AliasFor标签有几种使用方式。

### 同一注解使用

1，在同一个注解内显示使用；比如在@RequestMapping中的使用示例：

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {

    @AliasFor("path")
    String[] value() default {};

    @AliasFor("value")
    String[] path() default {};

    //...
}
```

又比如@ContextConfiguration注解中的value和locations属性：

```java
@Documented
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ContextConfiguration {
    @AliasFor("locations")
    String[] value() default {};

    @AliasFor("value")
    String[] locations() default {};
    //...
}
```

在同一个注解中成对使用即可，比如示例代码中，value和path就是互为别名。但是要注意一点，@AliasFor标签有一些使用限制，但是这应该能想到的，比如要求互为别名的属性属性值类型，默认值，都是相同的，互为别名的注解必须成对出现，比如value属性添加了@AliasFor("path")，那么path属性就必须添加@AliasFor("value")，另外还有一点，互为别名的属性必须定义默认值。

那么如果违反了别名的定义，在使用过程中就会报错，我们来做个简单测试：

```java
@ContextConfiguration(value = "aa.xml", locations = "bb.xml")
public class AnnotationUtilsTest {
    @Test
    public void testAliasfor() {
        ContextConfiguration cc = AnnotationUtils.findAnnotation(getClass(),
                ContextConfiguration.class);
        System.out.println(
                StringUtils.arrayToCommaDelimitedString(cc.locations()));
        System.out.println(StringUtils.arrayToCommaDelimitedString(cc.value()));
    }
}
```



![img](https://upload-images.jianshu.io/upload_images/807144-c68973e336530903.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/968/format/webp)

image.png

执行测试，报错；value和locations互为别名，不能同时设置；

稍微调整一下代码：

```java
@MyAnnotation
@ContextConfiguration(value = "aa.xml", locations = "aa.xml")
public class AnnotationUtilsTest {
```

或者

```java
@MyAnnotation
@ContextConfiguration(value = "aa.xml")
public class AnnotationUtilsTest {
```

运行测试，均打印出：

```java
aa.xml
aa.xml
```

### 覆盖元注解中的属性

2，显示的覆盖元注解中的属性；
先来看一段代码：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = AopConfig.class)
public class AopUtilsTest {
```

这段代码是一个非常熟悉的基于JavaConfig的Spring测试代码；假如现在我有个癖好，我觉得每次写@ContextConfiguration(classes = AopConfig.class)太麻烦了，我想写得简单一点，我就可以定义一个这样的标签：

```java
@Retention(RetentionPolicy.RUNTIME)
@ContextConfiguration
public @interface STC {

    @AliasFor(value = "classes", annotation = ContextConfiguration.class)
    Class<?>[] cs() default {};

}
```

1，因为@ContextConfiguration注解本身被定义为@Inherited的，所以我们的STC注解即可理解为继承了@ContextConfiguration注解；
2，我觉得classes属性太长了，所以我创建了一个cs属性，为了让这个属性等同于@ContextConfiguration属性中的classes属性，我使用了@AliasFor标签，分别设置了value（即作为哪个属性的别名）和annotation（即作为哪个注解）；

使用我们的STC：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@STC(cs = AopConfig.class)
public class AopUtilsTest {

    @Autowired
    private IEmployeeService service;
```

正常运行；
这就是@AliasFor标签的第二种用法，显示的为元注解中的属性起别名；这时候也有一些限制，比如属性类型，属性默认值必须相同；当然，在这种使用情况下，@AliasFor只能为作为当前注解的元注解起别名；

### 在一个注解中隐式声明别名

3，在一个注解中隐式声明别名；
这种使用方式和第二种使用方式比较相似，我们直接使用Spring官方文档的例子：

```java
 @ContextConfiguration
 public @interface MyTestConfig {

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] value() default {};

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] groovyScripts() default {};

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] xmlFiles() default {};
 }
```

可以看到，在MyTestConfig注解中，为value，groovyScripts，xmlFiles都定义了别名@AliasFor(annotation = ContextConfiguration.class, attribute = "locations")，所以，其实在这个注解中，value、groovyScripts和xmlFiles也互为别名，这个就是所谓的在统一注解中的隐式别名方式；

### 别名的传递

4，别名的传递；
@AliasFor注解是允许别名之间的传递的，简单理解，如果A是B的别名，并且B是C的别名，那么A是C的别名；
我们看一个例子：

```java
 @MyTestConfig
 public @interface GroovyOrXmlTestConfig {

    @AliasFor(annotation = MyTestConfig.class, attribute = "groovyScripts")
    String[] groovy() default {};

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] xml() default {};
 }
```

1，GroovyOrXmlTestConfig把 @MyTestConfig（参考上一个案例）作为元注解；
2，定义了groovy属性，并作为MyTestConfig中的groovyScripts属性的别名；
3，定义了xml属性，并作为ContextConfiguration中的locations属性的别名；
4，因为MyTestConfig中的groovyScripts属性本身就是ContextConfiguration中的locations属性的别名；所以xml属性和groovy属性也互为别名；
这个就是别名的传递性；

## 原理

明白@AliasFor标签的使用方式，我们简单来看看@AliasFor标签的使用原理；
首先来看看该标签的定义：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface AliasFor {

    @AliasFor("attribute")
    String value() default "";

    @AliasFor("value")
    String attribute() default "";

    Class<? extends Annotation> annotation() default Annotation.class;

}
```

可以看到，@AliasFor标签自己就使用了自己，为value属性添加了attribute属性作为别名；

那么就把这个注解放在我们需要的地方就可以了么？真就这么简单么？我们来做一个例子：
1，创建一个注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface MyAnnotation {

    @AliasFor("alias")
    String value() default "";

    @AliasFor("value")
    String alias() default "";

}
```

在该注解中，我们让alias和value属性互为别名；
2，完成测试类：

```java
@MyAnnotation(value = "aa", alias = "bb")
public class AnnotationUtilsTest {

    @Test
    public void testAliasfor2() {
        MyAnnotation ann = getClass().getAnnotation(MyAnnotation.class);
        System.out.println(ann.value());
        System.out.println(ann.alias());
    }
}
```

我们将MyAnnotation放在AnnotationUtilsTest上，可以看到，我们故意将value和alias值设置为不一样的，然后在测试代码中分别获取value()和alias()的值，结果打印：
aa
bb

WTF？和预期的不一样？原因很简单，AliasFor是Spring定义的标签，要使用他，只能让Spring来处理，修改测试代码：

```java
@MyAnnotation(value = "aa", alias = "bb")
public class AnnotationUtilsTest {
    
    @Test
    public void testAliasfor3() {
        MyAnnotation ann = AnnotationUtils.findAnnotation(getClass(),
                MyAnnotation.class);
        System.out.println(ann.value());
        System.out.println(ann.alias());
    }
}
```

这次我们使用Spring的AnnotationUtils工具类的findAnnotation方法来获取标签，然后再次打印value()和alias()值：



![img](https://upload-images.jianshu.io/upload_images/807144-2b7ace96ac12cc67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

image.png

如愿报错；所以，使用@AliasFor最需要注意一点的，就是只能使用Spring的AnnotationUtils工具类来获取；

而真正在起作用的，是AnnotationUtils工具类中的<A extends Annotation> A synthesizeAnnotation(A annotation, AnnotatedElement annotatedElement)方法；
这个方法传入注解对象，和这个注解对象所在的类型，返回一个经过处理（这个处理就主要是用于处理@AliasFor标签）之后的注解对象，简单说，这个方法就是把A注解对象---(经过处理)---->支持AliasFor的A注解对象，我们来看看其中的关键代码：

```java
DefaultAnnotationAttributeExtractor attributeExtractor =
    new DefaultAnnotationAttributeExtractor(annotation, annotatedElement);
InvocationHandler handler = 
    new SynthesizedAnnotationInvocationHandler(attributeExtractor);
return (A) Proxy.newProxyInstance(annotation.getClass().getClassLoader(),
    new Class<?>[] {(Class<A>) annotationType, SynthesizedAnnotation.class}, handler);
```

可以看到，本质原理就是使用了AOP来对A注解对象做了次动态代理，而用于处理代理的对象为SynthesizedAnnotationInvocationHandler；我们来看看SynthesizedAnnotationInvocationHandler中的重要处理代码：

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (isEqualsMethod(method)) {
        return annotationEquals(args[0]);
    }
    if (isHashCodeMethod(method)) {
        return annotationHashCode();
    }
    if (isToStringMethod(method)) {
        return annotationToString();
    }
    if (isAnnotationTypeMethod(method)) {
        return annotationType();
    }
    if (!isAttributeMethod(method)) {
        String msg = String.format("Method [%s] is unsupported for synthesized annotation type [%s]", method,
            annotationType());
        throw new AnnotationConfigurationException(msg);
    }
    return getAttributeValue(method);
}
```

在invoke（即拦截方法中，这个拦截方法就是在注解中获取属性值的方法，不要忘了，注解的属性实际上定义为接口的方法），其次判断，如果当前执行的方法不是equals、hashCode、toString、或者属性是另外的注解，或者不是属性方法，之外的方法（这些方法就是要处理的目标属性），都调用了getAttributeValue方法，所以我们又跟踪到getAttributeValue方法的重要代码：

```java
String attributeName = attributeMethod.getName();
        Object value = this.valueCache.get(attributeName);
        if (value == null) {
            value = this.attributeExtractor.getAttributeValue(attributeMethod);
```

这里我们重点关注的是如果没有缓存到值（这个先不用管），直接调用attributeExtractor.getAttributeValue方法获取属性值，那么，很容易猜到，如果属性有@AliasFor注解，就应该是这个方法在处理；那我们来看看这个方法又在做什么事情：

attributeExtractor是一个AnnotationAttributeExtractor类型，这个对象是在构造SynthesizedAnnotationInvocationHandler时传入的，默认是一个DefaultAnnotationAttributeExtractor对象；而DefaultAnnotationAttributeExtractor是继承AbstractAliasAwareAnnotationAttributeExtractor，看名字，真正的处理AliasFor标签的动作，应该就在这里面，于是继续看代码：

```java
public final Object getAttributeValue(Method attributeMethod) {
        String attributeName = attributeMethod.getName();
        Object attributeValue = getRawAttributeValue(attributeMethod);

        List<String> aliasNames = this.attributeAliasMap.get(attributeName);
        if (aliasNames != null) {
            Object defaultValue = AnnotationUtils.getDefaultValue(getAnnotationType(), attributeName);
            for (String aliasName : aliasNames) {
                Object aliasValue = getRawAttributeValue(aliasName);

                if (!ObjectUtils.nullSafeEquals(attributeValue, aliasValue) &&
                        !ObjectUtils.nullSafeEquals(attributeValue, defaultValue) &&
                        !ObjectUtils.nullSafeEquals(aliasValue, defaultValue)) {
                    throw new AnnotationConfigurationException(...)
                }
                if (ObjectUtils.nullSafeEquals(attributeValue, defaultValue)) {
                    attributeValue = aliasValue;
                }
            }
        }

        return attributeValue;
    }
```

对原代码做了些改造，但是我们能清晰的看到重点：
1，首先正常获取当前属性的值；
2，List<String> aliasNames = this.attributeAliasMap.get(attributeName);得到所有的标记为别名的属性名称；
3，Object aliasValue = getRawAttributeValue(aliasName);遍历获取所有别名属性的值；
4，三个重要判断，attributeValue、aliasValue、defaultValue相同，我们前面介绍的@AliasFor标签的传递性也是在这里体现；如果不相同，直接抛出异常；否则正常返回属性值；

至此，AliasFor的执行过程分析完毕；

## 小结

类似@AliasFor这样的注解，在Spring框架中比比皆是，而每一个这样的细节的点，都值得我们去体会。我们常常说Spring框架非常复杂，因为在每一个点的实现，都要考虑很多健壮性和扩展性的问题，这些，都是我们值得去研究的。



参考文档：

https://www.jianshu.com/p/869ed7037833