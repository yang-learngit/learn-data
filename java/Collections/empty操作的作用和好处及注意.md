# empty操作的作用和好处及注意

## empty操作的好处

1、如果你想 new 一个空的 List ，而这个 List 以后也不会再添加元素，那么就用 Collections.emptyList() 好了。
new ArrayList() 或者 new LinkedList() 在创建的时候有会有初始大小，多少会占用一内存。
每次使用都new 一个空的list集合，浪费就积少成多，浪费就严重啦，就不好啦
2、为了编码的方便。
比如说一个方法返回类型是List，当没有任何结果的时候，返回null,有结果的时候，返回list集合列表。
那样的话，调用这个方法的地方，就需要进行null判断。使用emptyList这样的方法，可以方便方法调用者。返回的就不会是null，省去重复代码。

 

## 注意的地方

这个空的集合是不能调用.add（），添加元素的。因为直接报异常。因为源码就是这么写的：直接抛异常。

哦，Collections里面没这么写，但是EmptyList继承了AbstractList这个抽象类，里面简单实现了部分集合框架的方法。
这里面的add方法最后调用的方法体，就是直接抛异常。
throw new UnsupportedOperationException();
这么解释add报异常就对啦。



下面简单看下这个源码：

```java
    /**
     * Collections 类里面的方法如下,一步步往下看就是啦
     */
    public static final <T> List<T> emptyList() {
        return (List<T>) EMPTY_LIST;
    }
	//。。。。。
    /**
     * Collections 类里面的方法如下,一步步往下看就是啦
     */
    public static final List EMPTY_LIST = new EmptyList<>();
	//。。。。。
	 /**
     * Collections里面的一个静态内部类
     */
    private static class EmptyList<E> extends AbstractList<E> implements RandomAccess, Serializable {
        private static final long serialVersionUID = 8842843931221139166L;
 
        public Iterator<E> iterator() {
            return emptyIterator();
        }
        public ListIterator<E> listIterator() {
            return emptyListIterator();
        }
 
        public int size() {return 0;}
        public boolean isEmpty() {return true;}
 
        public boolean contains(Object obj) {return false;}
        public boolean containsAll(Collection<?> c) { return c.isEmpty(); }
 
        public Object[] toArray() { return new Object[0]; }
 
        public <T> T[] toArray(T[] a) {
            if (a.length > 0)
                a[0] = null;
            return a;
        }
 
        public E get(int index) {
            throw new IndexOutOfBoundsException("Index: "+index);
        }
 
        public boolean equals(Object o) {
            return (o instanceof List) && ((List<?>)o).isEmpty();
        }
 
        public int hashCode() { return 1; }
 
        // Preserves singleton property
        private Object readResolve() {
            return EMPTY_LIST;
        }
    }

```

除了这个emptyList，之外，还有类似的，emptyMap，emptySet等等。具体看下图，都是一个套路。

![](https://github.com/yang-zhijiang/learn-data/blob/master/java/Collections/img/1-empty.png?raw=true)

