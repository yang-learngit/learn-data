# spring注解工具类AnnotatedElementUtils和AnnotationUtils

## 前言

spring为开发人员提供了两个搜索注解的工具类，分别是AnnotatedElementUtils和AnnotationUtils。在使用的时候，总是傻傻分不清，什么情况下使用哪一个。于是我做了如下的整理和总结。

## 二、AnnotationUtils官方解释

### 功能

　　用于处理注解，处理元注解，桥接方法（编译器为通用声明生成）以及超级方法（用于可选注解继承）的常规实用程序方法。请注意，JDK的内省工具本身并不提供此类的大多数功能。作为运行时保留注解的一般规则（例如，用于事务控制，授权或服务公开），始终在此类上使用查找方法（例如，findAnnotation（Method，Class），getAnnotation（Method，Class）和getAnnotations（方法））而不是JDK中的普通注解查找方法。您仍然可以在给定类级别的get查找（getAnnotation（Method，Class））和给定方法的整个继承层次结构中的查找查找（findAnnotation（Method，Class））之间明确选择。

### 术语

　　直接呈现，间接呈现和呈现的术语与AnnotatedElement的类级别javadoc中定义的含义相同（在Java 8中）。如果注解被声明为元素上存在的其他注解上的元注解，则注解在元素上是元存在的。如果A在另一个注解上直接存在或元存在，则注解A在另一个注解上存在元。

### 元注解支持

　　大多数find *（）方法和此类中的一些get *（）方法都支持查找用作元注解的注解。有关详细信息，请参阅此类中每个方法的javadoc。对于在组合注解中使用属性覆盖的元注解的细粒度支持，请考虑使用AnnotatedElementUtils的更具体的方法。

### 属性别名

　　此类中返回注解，注解数组或AnnotationAttributes的所有公共方法都透明地支持通过@AliasFor配置的属性别名。有关详细信息，请参阅各种synthesizeAnnotation *（..）方法。

### 搜索范围

　　一旦找到指定类型的第一个注解，此类中的方法使用的搜索算法将停止搜索注解。因此，将默默忽略指定类型的其他注解。



## AnnotatedElementUtils官方解释

### 功能

 　　用于在AnnotatedElements上查找注解，元注解和可重复注解的常规实用程序方法。AnnotatedElementUtils为Spring的元注解编程模型定义了公共API，并支持注解属性覆盖。如果您不需要支持注解属性覆盖，请考虑使用AnnotationUtils。请注意，JDK的内省工具本身不提供此类的功能。

### 注解属性覆盖

　　getMergedAnnotationAttributes（），getMergedAnnotation（），getAllMergedAnnotations（），getMergedRepeatableAnnotations（），findMergedAnnotationAttributes（），findMergedAnnotation（），findAllMergedAnnotations（）和findMergedRepeatableAnnotations（）的所有变体都支持组合注解中带有属性覆盖的元注解的方法。

### 查找与获取语义

　　**获取语义（Get semantics）**仅限于搜索AnnotatedElement上存在的注解（即本地声明或继承）或在AnnotatedElement上方的注解层次结构中声明的注解。
　　**查找语义（Find semantics）**更加详尽，提供了语义加上对以下内容的支持：

- 如果带注解的元素是类，则在接口上搜索
- 如果带注解的元素是类，则在超类上搜索
- 解析桥接方法，如果带注解的元素是方法
- 如果带注解的元素是方法，则在接口中搜索方法
- 如果带注解的元素是方法，则在超类中搜索方法

### 支持@Inherited

　　get语义之后的方法将遵循Java的@Inherited批注的约定，除了本地声明的注解（包括自定义组合注解）将优于继承注解。相反，查找语义之后的方法将完全忽略@Inherited的存在，因为查找搜索算法手动遍历类型和方法层次结构，从而隐式支持注解继承而不需要@Inherited。



## 准备两个测试的注解



```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@interface RequestMapping {

    String name() default "";

    @AliasFor("path")
    String[] value() default {};


    @AliasFor("value")
    String[] path() default {};
}


@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping
@interface PostMapping {

    @AliasFor(annotation = RequestMapping.class)
    String name() default "";

    @AliasFor(annotation = RequestMapping.class)
    String[] value() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] path() default {};
}
```



## 测试例子

### 父类拥有注解@RequestMapping，子类没有注解



```java
public class AnnotationTest {
    public static void main(String[] args) {
        System.out.println("ParentController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController getAnnotation @RequestMapping: " +AnnotationUtils.getAnnotation(ChildController.class, RequestMapping.class));
        System.out.println();

        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ChildController.class, RequestMapping.class));
        System.out.println();

        System.out.println("ParentController isAnnotated @RequestMapping: " + AnnotatedElementUtils.isAnnotated(ParentController.class, RequestMapping.class));
        System.out.println("ParentController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController isAnnotated @RequestMapping: " + AnnotatedElementUtils.isAnnotated(ChildController.class, RequestMapping.class));
        System.out.println("ChildController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ChildController.class, RequestMapping.class));
        System.out.println();

        System.out.println("ParentController hasAnnotation @RequestMapping: " + AnnotatedElementUtils.hasAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ParentController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController hasAnnotation @RequestMapping: " + AnnotatedElementUtils.hasAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ChildController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ChildController.class, RequestMapping.class));
    }
}

@RequestMapping(value = "parent/controller")
class ParentController {
}

class ChildController extends ParentController {
}
```



输出结果如下：

```java
ParentController getAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController getAnnotation @RequestMapping: null

ParentController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController isAnnotated @RequestMapping: true
ParentController getMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController isAnnotated @RequestMapping: false
ChildController getMergedAnnotation @RequestMapping: null

ParentController hasAnnotation @RequestMapping: true
ParentController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController hasAnnotation @RequestMapping: true
ChildController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
```



### 父类没有注解，子类有注解RequestMapping

父类没有注解，子类拥有注解@RequestMapping

输出结果如下：

```
ParentController getAnnotation @RequestMapping: null
ChildController getAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController findAnnotation @RequestMapping: null
ChildController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController isAnnotated @RequestMapping: false
ParentController getMergedAnnotation @RequestMapping: null
ChildController isAnnotated @RequestMapping: true
ChildController getMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController hasAnnotation @RequestMapping: false
ParentController findMergedAnnotation @RequestMapping: null
ChildController hasAnnotation @RequestMapping: true
ChildController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
```



### 父类有注解PostMapping，子类没有

父类拥有注解@PostMapping，子类没有注解

```java
public class AnnotationTest {
    public static void main(String[] args) {
        System.out.println("ParentController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController getAnnotation @RequestMapping: " +AnnotationUtils.getAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ParentController getAnnotation @PostMapping: " + AnnotationUtils.getAnnotation(ParentController.class, PostMapping.class));
        System.out.println("ChildController getAnnotation @PostMapping: " +AnnotationUtils.getAnnotation(ChildController.class, PostMapping.class));
        System.out.println();

        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ParentController findAnnotation @PostMapping: " + AnnotationUtils.findAnnotation(ParentController.class, PostMapping.class));
        System.out.println("ParentController findAnnotation @PostMapping: " + AnnotationUtils.findAnnotation(ChildController.class, PostMapping.class));
        System.out.println();

        System.out.println("ParentController isAnnotated @RequestMapping: " + AnnotatedElementUtils.isAnnotated(ParentController.class, RequestMapping.class));
        System.out.println("ParentController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ParentController isAnnotated @PostMapping: " + AnnotatedElementUtils.isAnnotated(ParentController.class, PostMapping.class));
        System.out.println("ParentController getMergedAnnotation @PostMapping: " + AnnotatedElementUtils.getMergedAnnotation(ParentController.class, PostMapping.class));
        System.out.println("ChildController isAnnotated @RequestMapping: " + AnnotatedElementUtils.isAnnotated(ChildController.class, RequestMapping.class));
        System.out.println("ChildController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ChildController isAnnotated @PostMapping: " + AnnotatedElementUtils.isAnnotated(ChildController.class, PostMapping.class));
        System.out.println("ChildController getMergedAnnotation @PostMapping: " + AnnotatedElementUtils.getMergedAnnotation(ChildController.class, PostMapping.class));
        System.out.println();

        System.out.println("ParentController hasAnnotation @RequestMapping: " + AnnotatedElementUtils.hasAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ParentController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ParentController hasAnnotation @PostMapping: " + AnnotatedElementUtils.hasAnnotation(ParentController.class, PostMapping.class));
        System.out.println("ParentController findMergedAnnotation @PostMapping: " + AnnotatedElementUtils.findMergedAnnotation(ParentController.class, PostMapping.class));
        System.out.println("ChildController hasAnnotation @RequestMapping: " + AnnotatedElementUtils.hasAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ChildController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ChildController hasAnnotation @PostMapping: " + AnnotatedElementUtils.hasAnnotation(ChildController.class, PostMapping.class));
        System.out.println("ChildController findMergedAnnotation @PostMapping: " + AnnotatedElementUtils.findMergedAnnotation(ChildController.class, PostMapping.class));
    }
}

@PostMapping(value = "parent/controller")
class ParentController {
}

class ChildController extends ParentController {
}
```



输出结果如下：

```java
ParentController getAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[], path=[])
ChildController getAnnotation @RequestMapping: null
ParentController getAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController getAnnotation @PostMapping: null

ParentController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[], path=[])
ChildController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[], path=[])
ParentController findAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController findAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController isAnnotated @RequestMapping: true
ParentController getMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ParentController isAnnotated @PostMapping: true
ParentController getMergedAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController isAnnotated @RequestMapping: false
ChildController getMergedAnnotation @RequestMapping: null
ChildController isAnnotated @PostMapping: false
ChildController getMergedAnnotation @PostMapping: null

ParentController hasAnnotation @RequestMapping: true
ParentController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ParentController hasAnnotation @PostMapping: true
ParentController findMergedAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController hasAnnotation @RequestMapping: true
ChildController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController hasAnnotation @PostMapping: true
ChildController findMergedAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])　　
```



### 父没注解，子有注解PostMapping

父类没有注解，子类拥有注解@PostMapping

输出结果如下。



```java
ParentController getAnnotation @RequestMapping: null
ChildController getAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[], path=[])
ParentController getAnnotation @PostMapping: null
ChildController getAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController findAnnotation @RequestMapping: null
ChildController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[], path=[])
ParentController findAnnotation @PostMapping: null
ChildController findAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController isAnnotated @RequestMapping: false
ParentController getMergedAnnotation @RequestMapping: null
ParentController isAnnotated @PostMapping: false
ParentController getMergedAnnotation @PostMapping: null
ChildController isAnnotated @RequestMapping: true
ChildController getMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController isAnnotated @PostMapping: true
ChildController getMergedAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])

ParentController hasAnnotation @RequestMapping: false
ParentController findMergedAnnotation @RequestMapping: null
ParentController hasAnnotation @PostMapping: false
ParentController findMergedAnnotation @PostMapping: null
ChildController hasAnnotation @RequestMapping: true
ChildController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[parent/controller], path=[parent/controller])
ChildController hasAnnotation @PostMapping: true
ChildController findMergedAnnotation @PostMapping: @com.hjzgg.apigateway.test.service.main.PostMapping(name=, value=[parent/controller], path=[parent/controller])
```



### PostMapping 注有 @RequestMapping

@PostMapping 注有 @RequestMapping，AnnotationUtils和AnnotatedElementUtils区别



```java
public class AnnotationTest {
    public static void main(String[] args) {
        System.out.println("ParentController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ChildController.class, RequestMapping.class));
        System.out.println();

        System.out.println("ParentController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ParentController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ChildController.class, RequestMapping.class));
    }
}

@RequestMapping(name = "parent", path="parent/controller")
class ParentController {
}

@PostMapping(name="child", value = "child/controller", consume = "application/json")
class ChildController extends ParentController {
}


@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@interface RequestMapping {

    String name() default "";

    @AliasFor("path")
    String[] value() default {};


    @AliasFor("value")
    String[] path() default {};

    String[] consume() default {};
}


@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping
@interface PostMapping {

    @AliasFor(annotation = RequestMapping.class)
    String name() default "";

    @AliasFor(annotation = RequestMapping.class)
    String[] value() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] path() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] consume() default "";
}
```



　　输出结果如下。



```
ParentController getAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController getAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[], path=[], consume=[])
ParentController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=, value=[], path=[], consume=[])

ParentController getMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController getMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=child, value=[child/controller], path=[child/controller], consume=[application/json])
ParentController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=child, value=[child/controller], path=[child/controller], consume=[application/json])
```



### PostMapping、RequestMapping各自独立

@PostMapping和@RequestMapping各自独立，AnnotationUtils和AnnotatedElementUtils区别



```
public class AnnotationTest {
    public static void main(String[] args) {
        System.out.println("ParentController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ChildController.class, RequestMapping.class));
        System.out.println();

        System.out.println("ParentController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ChildController.class, RequestMapping.class));
        System.out.println("ParentController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ParentController.class, RequestMapping.class));
        System.out.println("ChildController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ChildController.class, RequestMapping.class));
    }
}

@RequestMapping(name = "parent", path="parent/controller")
class ParentController {
}

@PostMapping(name="child", value = "child/controller", consume = "application/json")
class ChildController extends ParentController {
}


@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@interface RequestMapping {

    String name() default "";

    @AliasFor("path")
    String[] value() default {};


    @AliasFor("value")
    String[] path() default {};

    String[] consume() default {};
}


@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@interface PostMapping {

    @AliasFor(annotation = RequestMapping.class)
    String name() default "";

    @AliasFor(annotation = RequestMapping.class)
    String[] value() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] path() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] consume() default "";
}
```



　　输出结果如下。



```
ParentController getAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController getAnnotation @RequestMapping: null
ParentController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController findAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])

ParentController getMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController getMergedAnnotation @RequestMapping: null
ParentController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
ChildController findMergedAnnotation @RequestMapping: @com.hjzgg.apigateway.test.service.main.RequestMapping(name=parent, value=[parent/controller], path=[parent/controller], consume=[])
```



　　此时@PostMapping没注有@RequestMapping，所以AnnotationUtis.getAnnotation()和AnnotatedElementUtils.getMergeAnnotation()方法获取@RequestMapping信息为空，对应的find*()方法获取到的都是父类@RequestMapping信息。



## 相关方法解释

### AnnotationUtils.getAnnotation

　　从提供的AnnotatedElement获取annotationType的单个Annotation，其中注解在AnnotatedElement上存在或元存在。请注意，此方法仅支持单级元注解。要支持任意级别的元注解，请使用findAnnotation（AnnotatedElement，Class）。　　

### AnnotationUtils.findAnnotation

　　在提供的AnnotatedElement上查找annotationType的单个Annotation。如果注解不直接出现在提供的元素上，则将搜索元注解。

### AnnotatedElementUtils.isAnnotated

　　确定在提供的AnnotatedElement上或指定元素上方的注解层次结构中是否存在指定annotationType的注解。如果此方法返回true，则getMergedAnnotationAttributes方法将返回非null值。

### AnnotatedElementUtils.hasAnnotation

　　确定指定的annotationType的注解是否在提供的AnnotatedElement上或在指定元素上方的注解层次结构中可用。如果此方法返回true，则findMergedAnnotationAttributes方法将返回非null值。

### AnnotatedElementUtils.getMergedAnnotation

　　在提供的元素上方的注解层次结构中获取指定注解类型的第一个注解，将注解的属性与注解层次结构的较低级别中的注解的匹配属性合并，并将结果合成回指定注解类型的注解。完全支持@AliasFor语义，包括单个注解和注解层次结构。此方法委托给getMergedAnnotationAttributes（AnnotatedElement，Class）和AnnotationUtils.synthesizeAnnotation（Map，Class，AnnotatedElement）。

### AnnotatedElementUtils.findMergedAnnotation

　　在提供的元素上方的注解层次结构中查找指定注解类型的第一个注解，将注解的属性与注解层次结构的较低级别中的注解的匹配属性合并，并将结果合成回指定注解类型的注解。完全支持@AliasFor语义，包括单个注解和注解层次结构。





