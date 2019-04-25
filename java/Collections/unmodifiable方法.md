# unmodifiable操作方法

Collections.unmodifiableList方法，其实还有unmodifiableMap，unmodifiableSet两个相似的方法，接下来就分析一下。

unmodifiableMap,unmodifiableList,unmodifiableSet都是Collections的静态方法。可以明显看到三个方法都是unmodifiable开始的。

unmodifiable的中文意思是：不可更改，不可修改的。那么这三个方法到底有什么用呢？想必你已经有一个大概的猜测了。记住你的猜测，先来看一段代码：

## **使用示例**　　

```java
public void testUnmodifiable() {
        Map map = new HashMap();
        map.put("name", "zhangchengzi");
        map.put("age", 20);
        
        System.out.println("map before:"+map);//打印结果：map before:{name=zhangchengzi, age=20}
        
        Map unmodifiableMap = Collections.unmodifiableMap(map);
        System.out.println("unmodifiableMap before:"+unmodifiableMap);//打印结果：unmodifiableMap before:{name=zhangchengzi, age=20}。
        
        System.out.println("年龄："+unmodifiableMap.get("age"));//打印结果：年龄：20
        //修改年龄
        unmodifiableMap.put("age", 28);
        
        System.out.println("map after:"+map);
        System.out.println("unmodifiableMap after:"+unmodifiableMap);
}
```

相信你代码都看的很明白，那么后面两个为什么没有打印出结果呢？因为在13行抛出 了异常：

```java
java.lang.UnsupportedOperationException
    at java.util.Collections$UnmodifiableMap.put(Collections.java:1457)
    at com.test.learnmybatis.UserDaoTest.testUnmodifiable(UserDaoTest.java:63)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:497)
    at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:47)
    at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
    at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:44)
    at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
    at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:271)
    at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:70)
    at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:50)
    at org.junit.runners.ParentRunner$3.run(ParentRunner.java:238)
    at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:63)
    at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:236)
    at org.junit.runners.ParentRunner.access$000(ParentRunner.java:53)
    at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:229)
    at org.junit.runners.ParentRunner.run(ParentRunner.java:309)
    at org.eclipse.jdt.internal.junit4.runner.JUnit4TestReference.run(JUnit4TestReference.java:86)
    at org.eclipse.jdt.internal.junit.runner.TestExecution.run(TestExecution.java:38)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:538)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:760)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.run(RemoteTestRunner.java:460)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.main(RemoteTestRunner.java:206)
```



那么unmodifiableMap方法的作用就是将一个Map 进行包装，返回一个不可修改的Map。如果调用修改方法就会抛出java.lang.UnsupportedOperationException异常。

同样的道理unmodifiableList,unmodifiableSet就是将一个List或则Set进行包装，返回一个不可修改的List或者Set。你猜对了吗？那么unmodifiableMap是怎么做到这些事情的呢?

## unmodifiableMap方法源码解读　

　　上面提到过unmodifiableMap是Collections工具类的一个静态方法:　

```java
    /**
     * 返回一个指定Map的不可修改的视图，这个方法返回的视图为用户提供内部Map的"只读"访问，
     * 对是返回视图执行“读取”操作会直接作用到指定的Map,
     * 同时如果对返回视图执行修改操作（不论是直接的还是间接的）都会返回异常：UnsupportedOperationException
     * Returns an unmodifiable view of the specified map.  This method
     * allows modules to provide users with "read-only" access to internal
     * maps.  Query operations on the returned map "read through"
     * to the specified map, and attempts to modify the returned
     * map, whether direct or via its collection views, result in an
     * <tt>UnsupportedOperationException</tt>.<p>
     *
     * 如果指定的Map是可序列化的，则返回的Map也将是可序列化的。
     * The returned map will be serializable if the specified map
     * is serializable.
     *
     * @param <K> the class of the map keys
     * @param <V> the class of the map values
     * @param  m the map for which an unmodifiable view is to be returned.
     * @return an unmodifiable view of the specified map.
     */
    public static <K,V> Map<K,V> unmodifiableMap(Map<? extends K, ? extends V> m) {
        return new UnmodifiableMap<>(m);
    }
```

　　看源码的注释已经做了解释说明，但是还没有涉及到具体“不可修改的”原理，我们接着看源码。这个Collections.unmodifiableMap方法中，使用参数Map 实例化了一个UnmodifiableMap。我们看一下这个类：

```java
/**
     * @serial include
     */
    private static class UnmodifiableMap<K,V> implements Map<K,V>, Serializable {
        private static final long serialVersionUID = -1034234728574286014L;

        private final Map<? extends K, ? extends V> m;

        // 构造方法
        UnmodifiableMap(Map<? extends K, ? extends V> m) {
            if (m==null)
                throw new NullPointerException();
            this.m = m;
        }

        public int size()                        {return m.size();}
        public boolean isEmpty()                 {return m.isEmpty();}
        public boolean containsKey(Object key)   {return m.containsKey(key);}
        public boolean containsValue(Object val) {return m.containsValue(val);}
        public V get(Object key)                 {return m.get(key);}

        public V put(K key, V value) {
            throw new UnsupportedOperationException();
        }
        public V remove(Object key) {
            throw new UnsupportedOperationException();
        }
        public void putAll(Map<? extends K, ? extends V> m) {
            throw new UnsupportedOperationException();
        }
        public void clear() {
            throw new UnsupportedOperationException();
        }

        private transient Set<K> keySet;
        private transient Set<Map.Entry<K,V>> entrySet;
        private transient Collection<V> values;
      
        .....
    ｝
```



1，UnmodifiableMap类实现了Map接口，并且在这个类中有一个final修饰的 Map 类型的属性m。在构造方法中将调用Collections.unmodifiableMap(map)方法中传入的map实参，赋值给了UnmodifiableMap类的m属性。

2，上面在unmodifiableMap方法的注释中提到，对返回视图的修改，直接指向指定的map。为什么呢？看UnmodifiableMap的get方法，可以清晰看到，get方法直接到用了m.get(key)方法。

3，同时最关键的是“不可修改”是怎么实现的。看UnmodifiableMap的put方法，也可以很清晰的看到 在put方法中直接抛出了UnsupportedOperationException。

到这里Collections.unmodifiableMap方法的分析就进行完了，总结一下Collections.unmodifiableMap方法返回一个不可修改的Map。

## unmodifiableList,unmodifiableSet

unmodifiableList,unmodifiableSet的作用和实现原理和unmodifiableMap的是一样的，有兴趣就自己去看一下源码吧。