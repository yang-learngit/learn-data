

# Spring缓存源码剖析

Spring-Cache是Spring3.1引入的基于注解的缓存技术，本质上它并不是一个具体的缓存实现，而是一个对缓存使用的抽象，通过Spring AOP技术，在原有的代码上添加少量的注解来实现将这个方法转成缓存方法的效果。

## 介绍

Spring-Cache的大致结构如下图所示，最上层的接口有三个，`Cache`、`ValueWrapper`、`CacheManager`，分别对应的就是单个缓存、对缓存值的包装对象以及缓存管理器。

![](E:\我的\myGit\learn-data\spring\spring-cache\img\1-spring-cache.png)



接下来看一下这几个接口对应的源代码，可以说是非常简单的了：

```java
public interface Cache {
    /**
     * 获取该Cache的名称
	 */
    String getName();
    
    /**
     * 返回底层真是的缓存对象，接口中并不需要关系具体是怎么实现的，
     * 使用Map、Reids、Guava，Ecache等
	 */
    Object getNativeCache();
    
    // 这个是缓存根据key取value的方法，它返回的是value经过Spring Cache包装了之后的Object
    // 如果key对应的value不存在，直接返回 null
    // 从这里来看，Spring Cache是可以支持value is null的键值对缓存存储的，具体如何实现还是看后面的分析
    @Nullable
    ValueWrapper get(Object key);
    
    // 返回key所对应的value，并且这个value会被强转成给定的类型
    // 这个方法无法区分到底是缓存中存储的值是null还是在缓存中不存在这个值甚至可能是缓存中存在这个key所对应的value但却不是给定的type类型
    @Nullable
    <T> T get(Object key, @Nullable Class<T> type);
    
    // 同样的也是一个get方法，不同之处在于，它支持"if cached, return; otherwise create,
    // cache and return"这种模式，可以传入的函数式接口对象来实现
    @Nullabe
    <T> T get(Object key, Callable<T> valueLoader);
    
    // put方法，从这里的 @Nullable 注解来看，Spring-Cache是支持value is null的键值对缓存存储的
    void put(Object key, @Nullable Object value);
    
    @Nullable
    ValueWrapper putIfAbsent(Object key, @Nullable Object value);
    
    // 使这个key所对应的缓存失效
    void evict(Object key);
    
    // 清空所有的缓存
    void clear();

    // 这是Spring Cache用来对缓存的Value进行封装的接口
    @FunctionalInterface
    interface ValueWrapper {
        @Nullable
        Object get();
    }
}
```



`CacheManager`接口的代码就更简单啦，只有两个方法，它是用来对缓存进行管理的：

```java
public interface CacheManager {
    // 根据名字获取缓存对象
    @Nullable
    Cache getCache(String name);
    // 获取所有缓存的名字
    Collection<String> getCacheNames();
}
```



## `Cache`接口

首先我们来看有哪些类实现了`Cache`接口，如下图所示：
![cacheinterface](https://user-images.githubusercontent.com/16413289/41495024-71cc3900-7150-11e8-8608-cbb02d0725fd.png)

### NoOpCache

`NoOpCache`是一个专门用来禁用缓存的无操作实现，也就是说当你禁用缓存的时候，实际上是在使用这个缓存。所以它的实现里会比较简单，比如`get()`方法返回的是`null`，`put()`方法根本就不进行任何操作等。

```java
public class NoOpCache implements Cache {

	private final String name;


	/**
	 * Create a {@link NoOpCache} instance with the specified name.
	 * @param name the name of the cache
	 */
	public NoOpCache(String name) {
		Assert.notNull(name, "Cache name must not be null");
		this.name = name;
	}


	@Override
	public String getName() {
		return this.name;
	}

	@Override
	public Object getNativeCache() {
		return this;
	}

    // get方法返回null
	@Override
	@Nullable
	public ValueWrapper get(Object key) {
		return null;
	}

    // get方法返回null
	@Override
	@Nullable
	public <T> T get(Object key, @Nullable Class<T> type) {
		return null;
	}

	@Override
	@Nullable
	public <T> T get(Object key, Callable<T> valueLoader) {
		try {
			return valueLoader.call();
		}
		catch (Exception ex) {
			throw new ValueRetrievalException(key, valueLoader, ex);
		}
	}

    // put方法体为空
	@Override
	public void put(Object key, @Nullable Object value) {
	}

	@Override
	@Nullable
	public ValueWrapper putIfAbsent(Object key, @Nullable Object value) {
		return null;
	}

	@Override
	public void evict(Object key) {
	}

	@Override
	public void clear() {
	}

}
```



### `EhCacheCache`

`EhCacheCache`是对`EhCache`的一个封装，它就是实现了`Cache`接口，但是这些方法的具体实现都是调用了`EhCache`的API来实现的，比如看下面这份源码：

```java
public class EhCacheCache implements Cache {
    // 在这个类里面直接存储了一个来自于Ehcache的缓存对象，直接调用这个缓存对象提供的API
    private final Ehcache cache;
    // 构造方法可以说很简单了
    public EhCacheCache(EhCache ehcache) {
        // 如果ehcache是null的话，会直接抛运行时异常，说实话，之前没想到Assert可以这么用
        Assert.notNull(ehcache, "Ehcache must not be null");
        Status status = ehcache.getStatus();
        if (!Status.STATUS_ALIVE.equals(status)) {
            throw new IllegalArgumentException(
					"An 'alive' Ehcache is required - current cache is " + status.toString());
        }
        this.cache = ehcache;
    }
    
    @Override
    @Nullable
    public ValueWrapper get(Object key) {
        // 看一下lookup方法，Element是来自于Ehcache的对象，它是Ehcache对Value的封装
        Element element = lookup(key);
        // ValueWrapper是Spring Cache对Value的封装
        return toValueWrapper(element);
    }
    @Nullable
    private Element lookup(Object key) {
        // 看这里就是直接调用了Ehcache的API
        return this.cache(key);
    }
}
```



### `TransactionAwareCacheDecorator`

这是一个支持Spring的事务的Cache实现类，它支持同步的`put()`、`evict()`以及`clear()`，这些方法都会使用Spring的事务来管理，但是不支持`putIfAbsent()`这种类型的方法，来看一部分源码：

```java
public class TransactionAwareCacheDecorator implements Cache {
    private final Cache targetCache;
    // 可以看到，get方法其实是不需要事务来管理的
    @Override
    @Nullable
    public ValueWrapper get(Object key) {
        return this.targetCache.get(key);
    }
    // put方法则使用了Spring中的事务来进行管理
    @Override
    public void put(final Object key, @Nullable final Object value) {
        // Spring的事务支持开启或者关闭
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
			TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
				@Override
				public void afterCommit() {
					TransactionAwareCacheDecorator.this.targetCache.put(key, value);
				}
			});
		}
		else {
			this.targetCache.put(key, value);
		}
    }
}
```



### `AbstractValueAdaptingCache`

这是一个抽象类，主要的应用场景是：你需要在存储之前把这个key所对应的值做一份包装，比如`null`之类的特殊值，从这里来看，`CaffeineCache`是不支持null的值缓存的，来看一部分源码：

```java
public abstract class AbstractValueAdaptingCache implements Cache {
    // 这里看起来就非常明显了，基本上这个抽象类就是用来缓存值为null这种场景的
    // 如果这个值为true，那么来自于用户的null值这个对象会被替换为NullValue这个对象并存储
    // 当然真正返回给用户的时候，还是会被替换为null的
    // 这么做的原因是，如果都直接使用null对象，那么上层就无法判断这个key所对应的value在缓存中不存在
    // 还是它对应的value就是null。如果要辨别就只能选择去查一次数据库，这样就会出现某些特定的key可以直接
    // 穿透缓存的情况出现。而在这里使用NullValue包装了之后，上层就能够识别key对应的value为null的情况了
    private final boolean allowNullValues;
    // 还是看一下get方法吧，这里有lookup()和toValueWrapper()方法用来处理上面说到的情况
    @Override
    @Nullable
    public ValueWrapper get(Object key) {
        Object value = lookup(key);
        return toValueWrapper(value);
    }
    // 这个就简单的甩给子类来写了，因为每个子类使用的cache可能不一样
    @Nullable
    protected abstract Object lookup(Object key);
    // toValueWrapper()就是做一个很简单的封装
    @Nullable
    protected Cache.ValueWrapper toValueWrapper(@Nullable Object storeValue) {
        // 这里当storeValue为null的时候，表示从缓存中没有找到这个key所对应的value，所以直接返回null
        // 如果key所对应的value就是null的话，它在缓存中的应该是一个NullValue类型的对象，所以这里可以做一个区分
        // 而且做了区分之后，就把这个value的值包装成了SimpleValueWrapper
        return (storeValue != null ? new SimpleValueWrapper(fromStoreValue(storeValue)) : null);
    }
    // 这个方法的作用是把缓存中取出来的值转成用户真正需要的值
    @Nullable
    protected Object fromStoreValue(@Nullable Object storeValue) {
        if (this.allowNullValues && storeValue == NullValue.INSTANCE) {
            return null;
        }
        return storeValue;
    }

}
```



一些不支持`null`作为value缓存的、希望能避免缓存穿透的情况的`Cache`就继承了这个抽象类，比如`CaffeineCache`、`ConcurrentMapCache`以及`JCacheCache`这三个。

### `CaffeineCache`

`CaffeineCache`其实也没有什么内容，跟`EhCacheCache`差不多，这里是使用`CaffeineCache`的API来实现各种方法。

```java
public class CaffeineCache extends AbstractValueAdaptingCache {
    private final String name;
    private final com.github.benmanes.caffeine.cache.Cache<Object, Object> cache;
    // 它的get方法会奇怪一点，因为Caffeine底层是可以支持guava的，所以会有一个多的判断
    // 其他的方法也没啥好看的了
    @Override
    @Nullable
    public ValueWrapper get(Object key) {
        if (this.cache instanceof LoadingCache) {
            Object value = ((LoadingCache<Object, Object>)this.cache).get(key);
            return toValueWrapper(value);
        }
        return super.get(key);
    }
}
```



### ConcurrentMapCache

这就是一个非常简单的使用`ConcurrentMap`来实现的缓存：

```java
public class ConcurrentMapCache extends AbstractValueAdaptingCache {
    private final String name;
    private final ConcurrentMap<Object, Object> store;
    // 这是一个用来方便序列化的东西，同时它也用来决定是store by value还是store by reference
    @Nullable
    private final SerializationDelegate serialization;
    // 最简单的那个get()方法都懒得写，直接使用了父类的get方法，重写了lookup方法
    @Override
	@Nullable
	protected Object lookup(Object key) {
		return this.store.get(key);
	}
    // put方法其实也很简单
    @Override
    public void put(Object key, @Nullable Object value) {
        this.store.put(key, toStoreValue(value));
    }
    @Override
    protected Object toStoreValue(@Nullable Object userValue) {
        // 还是调用了父类的toStoreValue方法，就是上面说的那部分内容，比如如果是null，会被包装成其他对象
        Object storeValue super.toStoreValue(userValue);
        // 这里还有一部分跟序列化相关的代码
        if (this.serialization != null) {
            try {
                // store by value, a copy of each entry
                return serializeValue(this.serialization, storeValue);
            } cache (Throwable ex) {
                throw new IllegalArgumentException("Failed to serialize cache value '" + userValue +
						"'. Does it implement Serializable?", ex);
            }
        } else {
            return storeValue;
        }
    }
    // store by value的话，这里选择使用ByteArray的方式来存储value
    private Object serializeValue(SerializationDelegate serialization, Object storeValue) throws IOException {
		ByteArrayOutputStream out = new ByteArrayOutputStream();
		try {
			serialization.serialize(storeValue, out);
			return out.toByteArray();
		}
		finally {
			out.close();
		}
	}
}
```



### JCacheCache

`JCacheCache`更通用一些，它是调用了JSR107所规定的缓存的API来实现Spring的`Cache`接口所规定的方法的，来看一部分源码：

```java
public class JCacheCache extends AbstractValueAdaptingCache {
    // 这个Cache来自于javax.cache包，实现了JSR107标准
    private final javax.cache.Cache<Object, Object> cache;
    // 它的这些方法其实也是和上面的那些Cache类似的
    @Override
    @Nullable
    protected Object lookup(Object key) {
        return this.cache.get(key);
    }
    @Override
	public void evict(Object key) {
		this.cache.remove(key);
	}
    // 比较有意思的是这个get方法
    @Override
    @Nullable
    public <T> T get(Object key, Callable<T> valueLoader) {
        try {
            // 使用key来调用ValueLoaderEntryProcessor所带有的方法
            return this.cache.invoke(key, new ValueLoaderEntryProcessor<T>(), valueLoader);
        } catch (EntryProcessorException ex) {
			throw new ValueRetrievalException(key, valueLoader, ex.getCause());
		}
	}
    // ValueLoaderEntryProcessor长这样
    private class ValueLoaderEntryProcessor<T> implements EntryProcessor<Object, Object, T> {
        @SuppressWarnings("unchecked")
        @Override
        @Nullable
        public T process(MutableEntry<Object, Object> entry, Object...arguments) throws EntryProcessorException {
            Callable<T> valueLoader = (Callable<T>) arguments[0];
			if (entry.exists()) {
				return (T) fromStoreValue(entry.getValue());
			}
			else {
				T value;
				try {
					value = valueLoader.call();
				}
				catch (Exception ex) {
					throw new EntryProcessorException("Value loader '" + valueLoader + "' failed " +
							"to compute  value for key '" + entry.getKey() + "'", ex);
				}
				entry.setValue(toStoreValue(value));
				return value;
			}
        }
    }
}
```



## `CacheManager`接口

CacheManager简单描述就是用于管理Cache集合，并提供通过Cache名称获取对应Cache对象的方法，Cache用于存放具体的key-value值。举个栗子：一个Cache的名字是“奶牛厂”，那么这个Cache中可以根据“小白”获得叫做小白的奶牛，“小黑”获得叫做小黑点奶牛。

CacheManager`是Spring Cache用来管理缓存的接口，Spring Cache封装了很多不同的缓存，所以对应的也会有很多不同的`CacheManager`来对同一类型的缓存进行管理，其结构图如下：

![spring-cache-manager](https://user-images.githubusercontent.com/16413289/41605946-28374e52-7415-11e8-93b1-e42d646862b3.png)

可以来看一下源代码，非常简单，只有两个方法：

```java
/**
 * CacheManager可以通过名称来获取一个Cache对象
 * @author Costin Leau
 * @since 3.1
 */
public interface CacheManager {

	/**
	 * 通过name来获取Cache对象，从这里来看，每个Cache的name都必须是独一无二的
	 */
    @Nullable
	Cache getCache(String name);
	
	/**
     * 返回这个CacheManager管理的Cache集合的所有Cache名称
	 */
	Collection<String> getCacheNames();
}
```



### NoOpCacheManager

`NoOpCacheManager`是用来支持禁用缓存的无操作缓存实现管理类，适用于管理声明了缓存但是没有一个真实的存储实现的那些缓存，源码其实是很简单的啦：

```java
public NoOpCacheManager implements CacheManager {
    // 这里用了一个 map 来管理缓存对象，key为缓存对象的名称，value为缓存对象本身
    private final ConcurrentMap<String, Cache> caches = new ConcurrentHashMap<>(16);
    // 感觉这里没什么必要存缓存的名字，直接从上面的Map里取就行了，
    // 也许是考虑到获取缓存名字列表是一个高频的操作吧，所以这里保存了下来
    private final Set<String> cacheNames = new LinkedHashSet<>(16);
    // 这个方法看起来还是有点奇怪的，它永远都返回一个NoOpCache，而不是返回null
    // 即使对应的name的cache不存在，也会新创建一个NoOpCache并存下(感觉上是为了支持Runtime的时候动态的创建缓存，在后面的一些CacheManager中也会有dynamic创建缓存的特性)
    // 然而又标上了@Nullable，简直可怕
    // 比较有意思的还是synchronized关键字，给操作加了个锁
    @Override
    @Nullable
    public Cache getCache(String name) {
        Cache cache = this.caches.get(name);
        if (cache == null) {
            this.caches.putIfAbsent(name, new NoOpCache(name));
            synchronized (this.cacheNames) {
                this.cacheNames.add(name);
            }
        }
        return this.caches.get(name);
    }
    // 也是给操作加锁，因为获取的同时可能也有其他线程正在往里增加缓存的名字
    @Override
    public Collection<String> getCacheNames() {
        synchronized (this.cacheNames) {
            return Collections.unmodifiableSet(this.cacheNames);
        }
    }
}
```



### CaffeineCacheManager

`CaffeineCacheManager`里面会有一些`Caffeine`的API，用来创建和刷新缓存实例，这里就不多做介绍了，专注于介绍Spring Cache所约定的需要实现的方法：

```java
public CaffeineCacheManager implements CacheManager {
    // 按照惯例我们还是使用一个map来存储缓存实例
    // 然后这里很分裂的居然没有用set来存储缓存名称，之前的那个NoOpCacheManager里用set来存储缓存名称确实是多余
    private final ConcurrentMap<String, Object> cacheMap = new ConcurrentHashMap<>(16);
    // 对应的getCacheNames方法就很简单了，直接通过cacheMap拿到keySet
    @Override
    public Collection<String> getCacheNames() {
        return Collections.unmodifiableSet(this.cacheMap.keySet());
    }
    // 然后这里使用了一个dynamic的字段来控制是否在Runtime的时候创建缓存实例
    // cacheBuilder是一个用来创建CaffeineCache实例对象的builder，私以为这里提供CaffeineCacheCache的builder会让整个设计意图更清晰一些
    // cacheLoader是用来加载缓存值用的
    private boolean dynamic = true;
    private Caffeine<Object, Object> cacheBuilder = Caffeine.newBuilder();
    @Nullable
    private CacheLoader<Object, Object> cacheLoader;
    // 默认支持null值的存储
    private boolean allowNullValues = true;
    // Spring Cache把创建缓存的方式分为static和dynamic两种
    // 如果调用过这个方法，就意味着使用了static的方式来创建缓存，
    // 无法再动态(dynamic)的创建Cache，这么做的意图应该是
    // 希望使用者能够明确到底是动态的创建缓存还是一次性全部创建完毕，不支持两者混用的模式
    public void setCacheNames(@Nullable Collection<String> cacheNames) {
        if (cacheNames != null) {
            for (String name : cacheNames) {
                this.cacheMap.put(name, createCaffeineCache(name));
            }
            this.dynamic = false;
        } else {
            this.dynamic = true;
        }
    }
    // getCache方法当然也要照顾到dynamic创建cache的需求啦
    @Override
    @Nullable
    public Cache getCache(String name) {
        Cache cache = this.cacheMap.get(name);
        // 这里一定是需要判断当前是否支持dynamic创建缓存
        if (cache == null && this.dynamic) {
            synchronized (this.cacheMap) {
                // 这里需要再重新判断一遍这个name对应的Cache存不存在
                // 主要是考虑到多线程并发的环境下，有可能其他的线程已经创建了这个Cache
                cache = this.cacheMap.get(name);
                if (cache == null) {
                    cache = createCaffeineCache(name);
                    this.cacheMap.put(name, cache);
                }
            }
        }
        return cache;
    }
}
```

### CompositeCacheManager

这是一个能够把很多`CacheManager`的实现组合起来的`CacheManager`实现，通过一个List来存储很多的`CacheManager`实例，并且通过遍历这些实例的方式来做某些操作。
因为它还实现了`InitializingBean`接口，所以Spring在初始化这个Bean的时候，会自动调用afterPropertiesSet方法。

```java
public class CompositeCacheManager implements CacheManager, InitializingBean {
    // 使用了一个List来存储众多的CacheManager实例
    private final List<CacheManager> cacheManagers = new ArrayList<>();
    // 这个字段的作用是，决定是否往cacheManagers这个List添加一个NoOpCacheManager
    // 如果有NoOpCacheManager存在的话，所有的getCache的请求里某个name没有对应的Cacha实例
    // 也会被NoOpCacheManager所捕获，然后getCache方法的返回就是NoOpCache实例而不是null
    // (可以参看NoOpCacheManager的实现，它那个实现可以支持即使你输入的name对应的Cache不存在，它也可以给你创建要给NoOpCache)
    // 即是否为某个name所对应的CachaManager实例不存在时创建要给NoOpCacheManager实例，
    private boolean fallbackToNoOpCache = false;
    // getCache和getCacheNames方法
    @Override
    @Nullable
    public Cache getCache(String name) {
        for (CacheManager cacheManager : this.cacheManagers) {
            Cache cache = cacheManager.get(name);
            if (cache != null) {
                return cache;
            }
        }
        return null;
    }
    @Override
    public Collection<String> getCacheNames() {
        Set<String> names = new LinkedHashSet<>();
        for (CacheManager manager : this.cacheManagers) {
            names.addAll(manger.getCacheNames());
        }
        return Collections.unmodifiableSet(names);
    }
    // 这个方法来自于InitializlingBean的实现，这里自动创建了一个NoOpCacheManager并且加到了cachaManagers这个列表里面
    @Override
    public void afterPropertiesSet() {
        if (this.fallbackToNoOpCache) {
            this.cacheManagers.add(new NoOpCacheManager());
        }
    }
}
```

### TransactionAwareCacheManagerProxy

这是一个对`CacheManager`的封装，这个`CacheManager`封装了能够支持事务的`Cache`(使用Spring的事务来对`put`之类的方法进行原子性操作的`Cache`)。

```java
public class TransactionAwareCacheManagerProxy implements CacheManager, InitializingBean {
    @Nullable
    private CacheManager targetCacheManager;
    // 这个还是Spring的那个bean初始化的方法
    @Override
    public void afterPropertiesSet() {
        if (this.targetCacheManager == null) {
            throw new IllegalArgumentException("Property 'targetCacheManager' is required");
        }
    }
    @Override
    @Nullable
    public Cache getCache(String name) {
        Assert.state(this.targetCacheManager != null, "No target CacheManager set");
        Cache targetCache = this.targetCacheManager.getCache(name);
        // 所以它实际上就是调用了TransactionAwareCacheDecorator对cache进行了封装
        // 也就是说Spring Cache支持在Cache层以及CacheManager层对Cache进行事务的封装
        return (targetCache != null ? new TransactionAwareCacheDecorator(targetCache) : null);
    }
    @Override
	public Collection<String> getCacheNames() {
		Assert.state(this.targetCacheManager != null, "No target CacheManager set");
		return this.targetCacheManager.getCacheNames();
	}

}
```

### ConcurrentMapCacheManager

```java
/**
 * ConcurrentMapCacheManager负责管理ConcurrentMapCache对象，支持懒加载获取，也支持预先实例化对象，
 * 通过调用#setCacheNames方法可以预定义一些ConcurrentMapCache到ConcurrentMapCacheManager中，
 * 但是一旦设置过后，dynamic就会设置为false，这时就不再支持动态创建Cache功能。
 *
 * 通常是否懒加载可以通过不同的构造器来控制，new ConcurrentMapCacheManager()创建懒加载的CacheManager，
 * new ConcurrentMapCacheManager(String... cacheNames)创建预先定义Cache的CacheManager。
 */
public class ConcurrentMapCacheManager implements CacheManager, BeanClassLoaderAware {
    // 使用ConcurrentMap来管理Cache集合对象
    // 支持动态的创建缓存，支持Null Value，支持按值存储以及按引用存储
    private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);
    //表示是否可以动态创建Cache缓存，如果指定过name那么就不能动态创建
    private boolean dynamic = true;
    private allowNullValues = true;
    private boolean storeByValue = false;
    @Nullable
    private SerializationDelegate serialization;
    /**
     * 这里会初始化ConcurrentMapCache，并存放到ConcurrentMap中进行管理
     * 一旦初始化过，这dynamic=false，在调用getCache(name)是就不会动态创建了
	 */
    public void setCacheNames(@Nullable Collection<String> cacheNames) {
		if (cacheNames != null) {
			for (String name : cacheNames) {
				this.cacheMap.put(name, createConcurrentMapCache(name));
			}
			this.dynamic = false;
		}
		else {
			this.dynamic = true;
		}
	}
    // 重点来看一下storeByValue
    public void setStoreByValue(boolean storeByValue) {
        if (storeByValue != this.storeByValue) {
            this.storeByValue = storeByValue;
            // 重新创建Cache
            recreateCaches();
        }
    }
    private void recreateCaches() {
        for (Map.Entry<String, Cache> entry : this.cacheMap.entrySet()) {
            entry.setValue(createConcurrentMapCache(entry.getKey()));
        }
    }
    protected Cache createConcurrentMapCache(String name) {
        // 根据前面的ConcurrentMapCache的分析可知，store by value的时候，就是穿一个不为空的SerializationDelegate对象进去即可
        SerializationDelegate actualSerialization = (isStoreByValue() ? this.serialization : null);
		return new ConcurrentMapCache(name, new ConcurrentHashMap<>(256),
				isAllowNullValues(), actualSerialization);
    }
    // 这个方法来自于BeanClassLoaderAware接口，它也称为回调接口
    // 可以让受管Bean本身知道它是由哪一类装载器负责装载的，这里主要用来控制store by value or store by reference
    @Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		this.serialization = new SerializationDelegate(classLoader);
		// Need to recreate all Cache instances with new ClassLoader in store-by-value mode...
		if (isStoreByValue()) {
			recreateCaches();
		}
	}
    
    @Override
	public Collection<String> getCacheNames() {
		return Collections.unmodifiableSet(this.cacheMap.keySet());
	}
    
    /**
	 * 根据缓存name来获取关联的ConcurrentMapCache实例
	 * 如果ConcurrentMapCacheManager中没有获取到，则动态获取。（能否动态获取需要看dynamic是否为true）
	 */
	@Override
	@Nullable
	public Cache getCache(String name) {
		Cache cache = this.cacheMap.get(name);
		if (cache == null && this.dynamic) {
			synchronized (this.cacheMap) {
				cache = this.cacheMap.get(name);
				if (cache == null) {
					cache = createConcurrentMapCache(name);
					this.cacheMap.put(name, cache);
				}
			}
		}
		return cache;
	}
}
```

### AbstractCacheManager

这是一个比较通用的`CacheManager`实现，比较适用于缓存创建之后就不会被改变的运行环境。

AbstractCacheManager提供了基本的操作，如果已经存在的CacheManager无法满足使用要求，可以继承AbstractCacheManager类实现自己的CacheManager。

```java
public AbstractCacheManager implements CacheManager, InitializingBean {
    private final ConcurrentMap<String, Object> cacheMap = new ConcurrentHashMap<>(16);
    // 这里使用了volatile保证线程安全，Volatile的应用场景是非常有限的一组用例：多个变量之间或者某个变量的当前值与修改后值之间没有约束。
    // 正好满足这里的应用场景：缓存创建之后不会被改变
    // 而且getCacheNames()方法就不多说了，直接返回这个field的值即可
    private volatile Set<String> cacheNames = Collections.emptySet();
    // 用于初始化，实际上是根据配置创建各种Cache，然后创建一个CacheManager实例
    @Override
    public void afterPropertiesSet() {
        initializeCaches();
    }
    public void initializeCaches() {
        Collection<? extends Cache> caches = loadCaches();
        // 同样，在创建的过程中，要是线程安全的，所以这里加了锁
        synchronized (this.cacheMap) {
            // 这一句是不是感觉没什么必要啊
            this.cacheNames = Collections.emptySet();
            this.cacheMap.clear();
            Set<String> cacheNames = new LinkedHashSet<>(caches.size());
            for (Cache cache : caches) {
                String name = cache.getName();
                // 这里可能会对获取到的Cache实例做一层封装，
                // 这个deocrateCache方法可以由子类来重写，加上自己独有的逻辑
                this.cacheMap.put(name, decorateCache(cache));
                cacheNames.add(name);
            }
            this.cacheNames = Collections.unmodifiableSet(cacheNames);
        }
    }
    // 加载配置中定义并创建的Cache，这个肯定是由子类来实现的
    protected abstract Collection<? extends Cache> loadCaches();

    // 即根据Cache名称获取与之对应的Cache，如果没有找到对应的Cache，则会调用getMissingCache(String)，默认getMissingCache返回null。将决定权交给实现者，你可以创建一个Cache，或者记录日志。
    @Override
    @Nullable
    public Cache getCache(String name) {
        Cache cache = this.cacheMap.get(name);
        if (cache != null) {
            return cache;
        } else {
            // 考虑到多线程并发的运行环境，这里采用了多次寻找cache的方式，并且加了锁
            synchronized (this.cacheMap) {
                cache = this.cacheMap.get(name);
                if (cache == null) {
                    // getMissingCache方法在这个抽象类里面是直接返回null了
                    // 但是子类可以重写这个方法并增加自己的特有逻辑，比如某些时候为了程序不报错或者兼容
                    // 传入value为null的name，可以这个方法里面返回一个默认的NoOpCache
                    cache = getMissingCache(name);
                    if (cache != null) {
                        cache = decorateCache(cache);
                        this.cacheMap.put(name, cache);
                        // 这里就需要更新cacheNames这个Set了，这个updateCacheNames的方法是很简单的啦
                        // 如果getMissingCache后cache不为空，这里会调用updateCacheNames方法，更新cacheNames集合。cacheNames是一个只读的Set，每次更新需要重新创建新的Set。
                        updateCacheNames(name);
                    }
                }
                return cache;
            }
        }
    }
    
    // 根据一个Cache名称得到对应的Cache，如果没有就返回null，不会触发getMissingCache方法。
    @Nullable
	protected final Cache lookupCache(String name) {
		return this.cacheMap.get(name);
	}
    
    // 加入getMissingCache方法创建了Cache的实例，则会调用decorateCache方法对原有的Cache进行一次包装，这个通过方法名字应该可以猜到可能会用到[修饰模式]（也有叫装饰模式等），这里也没有给出具体实现。
    protected Cache decorateCache(Cache cache) {
		return cache;
	}
}
```

afterPropertiesSet()方法：来自实现的org.springframework.beans.factory.InitializingBean接口，在Bean实例化之后调用。这里使用了[模板方法模式](https://zh.wikipedia.org/wiki/%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95)，loadCaches()方法的实现交给具体的子类，大致意思就是：这里需要得到Cache的集合，具体这个Cache集合从哪里来，具体的Cache的实现类是什么一概不管。





实际上还会有一个`SimpleCacheManager`继承了这个抽象类，并实现了一个简单的缓存管理器，但是太简单了且没有什么实用价值，所以这篇文章里就不提了。

### `AbstractTransactionSupportingCacheManager`

这是一个为了支持内置的Spring管理的事务的特性的`CacheManager`基础实现，可以通过`transactionAware`这个字段来控制是否要启用事务管理。

```java
public abstract class AbstractTransactionSupportingCacheManager extends AbstractCacheManaget {
    // 用来控制是否要启用事务管理，默认不启用
    private boolean transactionAware = false;
    // 通过decorateCache方法来对Cache进行封装
    @Override
    protected Cache decorateCache(Cache cache) {
        return (isTransactionAware() ? new TransactionAwareCacheDecorator(cache) : cache);
    }
}
```



### `EhCacheCacheManager`

这就是一个用来管理`EhCache`的`CacheManager`实现，也没太多东西：

```java
public class EhCacheCacheManager extends AbstractTransactionSupportingCacheManager {
    // 其实是对ehcache自导的CacheManager做了一层封装
    @Nullable
    private net.sf.ehcache.CacheManager cacheManager;
    @Override
    public void afterPropertiesSet() {
        if (getCacheManager() == null) {
            // 创建Ehcache自带的cacheManager
			setCacheManager(EhCacheManagerUtils.buildCacheManager());
		}
		super.afterPropertiesSet();
    }
    // 这个是子类肯定要实现的，基本上是创建Cache实例的核心方法
    @Override
    protected Collection<Cache> loadCaches() {
        net.sf.ehcache.CacheManager cacheManaget = getCacheManager();
        Assert.state(cacheManager != null, "No CacheManager set");
        Status status = cacheManager.getStatus();
		if (!Status.STATUS_ALIVE.equals(status)) {
			throw new IllegalStateException(
					"An 'alive' EhCache CacheManager is required - current cache is " + status.toString());
		}

		String[] names = getCacheManager().getCacheNames();
		Collection<Cache> caches = new LinkedHashSet<>(names.length);
		for (String name : names) {
			caches.add(new EhCacheCache(getCacheManager().getEhcache(name)));
		}
		return caches;
    }
    // getMissingCache，如果对应的Cache实例不存在的时候，这里会再次检查一遍，如果肯定没有，返回的还是null
    @Override
	protected Cache getMissingCache(String name) {
		net.sf.ehcache.CacheManager cacheManager = getCacheManager();
		Assert.state(cacheManager != null, "No CacheManager set");

		// Check the EhCache cache again (in case the cache was added at runtime)
		Ehcache ehcache = cacheManager.getEhcache(name);
		if (ehcache != null) {
			return new EhCacheCache(ehcache);
		}
		return null;
	}
}
```

### `JCacheCacheManager`

这个就是对`javax.cache.CacheManager`的封装，也非常简单，和`EhCacheCacheManager`类似，主要的三个核心的方法都是一样的实现逻辑：

```java
public class JCacheCacheManager extends AbstractTransactionSupportingCacheManager {
	@Nullable
	private javax.cache.CacheManager cacheManager;
	private boolean allowNullValues = true;

	@Override
	public void afterPropertiesSet() {
		if (getCacheManager() == null) {
			setCacheManager(Caching.getCachingProvider().getCacheManager());
		}
		super.afterPropertiesSet();
	}

	@Override
	protected Collection<Cache> loadCaches() {
		CacheManager cacheManager = getCacheManager();
		Assert.state(cacheManager != null, "No CacheManager set");

		Collection<Cache> caches = new LinkedHashSet<>();
		for (String cacheName : cacheManager.getCacheNames()) {
			javax.cache.Cache<Object, Object> jcache = cacheManager.getCache(cacheName);
			caches.add(new JCacheCache(jcache, isAllowNullValues()));
		}
		return caches;
	}

	@Override
	protected Cache getMissingCache(String name) {
		CacheManager cacheManager = getCacheManager();
		Assert.state(cacheManager != null, "No CacheManager set");

		// Check the JCache cache again (in case the cache was added at runtime)
		javax.cache.Cache<Object, Object> jcache = cacheManager.getCache(name);
		if (jcache != null) {
			return new JCacheCache(jcache, isAllowNullValues());
		}
		return null;
	}

}
```



## SpringCacheAnnotations(注解)

Spring Cache的注解有这几个：

- `@EnableCaching`：用于开启注释驱动的Spring Cache
- `@CacheEvict`：用于清空缓存
- `@Cacheable`：用于表明这个方法or类中的所有方法都是支持缓存的
- `@CachePut`：用于往缓存中存数据
- `@Caching`： 用于在一个方法或者类上同时指定多个Spring Cache相关的注解
- `@CacheConfig`：用于自定义的对某个方法or类的缓存进行配置

其实我们可以对这些注解做一些分类的：

- `@EnableCaching`、`@Caching`和`@CacheConfig`都属于配置的注解
- `@CachePut`、`CacheEvict`和`@Cacheable`都属于具体缓存操作的注解

先来看一下annotation这个package下的Java class diagrams，都是与annotation相关的类：

先来看一下annotation这个package下的Java class diagrams，都是与annotation相关的类：
![spring-cache-annotations](https://user-images.githubusercontent.com/16413289/41819989-dca67242-77fc-11e8-9c4f-34f8e3a623c1.png)

可以很明显的从名字上看出来，有一些配置类(`ProxyCachingConfiguration`等)也有一些用来parse注解的类(`SpringCacheAnnotationParser`)，还有一些对注解的操作进行封装的类(`AnnotationCacheOperationSource`)，然后剩下的就是几个注解了。



### 配置的注解

我们先从Spring Cache自带的几个注解类的代码开始看起，主要看这些注解有什么功能，从源代码来反推这些注解的用法，并且也会分析一些在这些代码中出现的类。

#### @EnableCaching

```java
// 指定这个注解用来修饰的范围是类、接口(包括注解类型)或enum声明
@Target(ElementType.TYPE)
// 运行时有效，可以在运行时通过反射获取该注解的属性值，从而来实现一些业务逻辑
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
  // 表明使用CGLIB基于继承的代理还是JDK标准的基于接口的代理，默认是后者(false)
  // 而且这个属性只有在mode被设置为AdviceMode.PROXY的时候才有用
  // 需要注意的是，如果设置true，这个作用域是全局的，而不仅仅只是影响到@Cacheable这种注解
  // 比如你可能在其他地方使用了@Transactional注解，它的代理模式也会同时被更新成使用CGLIB的代理
  // 当然这种设定在实践中并不会产生什么负面影响，除非你希望明确的使用两种代理，并且希望做一个对比
  boolean proxyTargetClass() default false;
  // 这个是用来选择AOP的实现方式的，默认是使用代理，也可以选择ASPECTJ
  AdviceMode mode() default AdviceMode.PROXY;
  // 当遇到多个切面在同一个切点的时候，这个order用来决定切面的执行顺序
  // 一般来说，值越大执行的顺序越靠后
  int order() default Ordered.LOWEST_PERCEDENCE;
}
```



#### @CacheEvict

用于删除某条缓存记录的注解：

```java
// 可用于方法、类、接口、枚举类声明
@Target({ElementType.METHOD, ElementType.Type})
@Retention(RententionPolicy.RUNTIME)
// 表明这个注解是允许被继承的
@Inherited
@Documented
public @interface CacheEvict {
  // 居然还有@AliasFor这种东西。。。
  @AliasFor("cacheNames")
  String[] value() default {};
	@AliasFor("value")
	String[] cacheNames() default {};
  // key可以是固定值，也可以是SpEL表达式，所以用String来表示，但是我们需要解析它
  String key() default "";
  // 自定义cacheManager的名字，它会用于创建一个CacheResolver对象
  // 如果没有设置，则会创建一个默认的CacheResolver对象
  String cacheManager() default "";
  // 自定义key生成策略的Bean的名字，这是一个新的没出现过的概念(key生成器)，如果想要详细的了解，可以看后面的几个章节
  String keyGenerator() default "";
  // 自定义的CacheResolver的名字，CacheResolver是用来判断在拦截这个方法调用的时候，
  // 到底使用哪个Cache实例来做操作。举个例子，你有一个方法MethodA使用了@Cacheable注释，然后执行之前
  // 被SpringCache的AOP拦截了，这时候需要能够找到一个准确的Cache实例来进行缓存相关的操作
  // CacheResolver就是用来保证找到这个Cache实例的方法，当然这里可以使用指定名字的方式来寻找这个Cache实例
  // 如果想要详细的了解，可以看后面的几个章节
  String cacheResolver() default "";
  // SpEL表达式，用来表示满足删除缓存条目的条件
  String condition() default "";
  // 是否一处缓存里的所有条目，默认的话是只移除对应key的者一条缓存记录
  boolean allEntries() default false;
  // 是否应该在调用这个函数之前删除缓存条目，默认是在调用函数之后再删除缓存
  boolean beforeInvocation() default false;
}
```



#### @CachePut

`@CachePut`里的属性和`@CacheEvict`基本上差不多，只是多了一个`unless`：

```java
public @interface CachePut {
  // 这是一个SpEL表达式，用来否决缓存放置操作
  // 它和condition是有区别的
  String unless() default "";
}
```



#### @Cacheable

对比于`@CachePut`，`@Cacheable`多一个`sync`的属性：

```java
public @interface Cacheable {
  // 用来决定被标记的方法是否再多线程环境中只能同步(加锁)的被调用，默认不是
  boolean sync() default false;
}
```



#### @Caching

`@Caching`是用来对多个注解进行组合的注解：

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Caching {
  Cachable[] cacheable() default {};
  CachePut[] put() default {};
  CacheEvict[] evict() default {};
}
```



#### @CacheConfig

这个注解是用于在同一个类中共享一些基础的cache配置的：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CacheConfig {
  String[] cacheNames() default {};
	String keyGenerator() default "";
	String cacheManager() default "";
	String cacheResolver() default "";
}
```



### 缓存操作注解

使用了`BasicOperation`来封装缓存的操作行为。
看下面的类图结构，`CacheOperation`部分是Spring Cache自己的缓存操作封装，`JCacheOperation`是为了支持JSR-107而做的对缓存操作的封装。
![basicoperation](https://user-images.githubusercontent.com/16413289/42723997-b17e54fe-879c-11e8-894a-066c23b93443.png)

其实从类图中可以看出来，结构确实是非常清晰明白的，除去`CacheOperationInvocationContext`之外，把缓存操作按照这个结构来划分：

- Spring Cache自带的缓存操作
  - `@CachePut`、`CacheEvict`和`@Cacheable`三个注解对应的Operation
- JSR-107要求的缓存操作
  - 不需要key的操作，`@CacheRemoveAll`对应的Operation
  - 需要key的操作，`@CacheResult`、`@CachePut`、`@CacheRemove`对应的Operation

`BasicOperation`本身很简单：

```
public interface BasicOperation {
  // 很简单的只有一个拿到所有缓存名的方法
  Set<String> getCacheNames();
}
```



来看一下实现了这个接口的其他实例。

# `CacheOperation`

回想一下在Spring Cache的注解中出现过的、可以配置的东西有cacheName、key、keyGenerator、cacheManager、cacheResolver、condition等等。`CacheOperation`是对这些操作的封装，所以它肯定是带有这些信息的，源码如下:

```java
public abstract class CacheOperation implements BasicOperation {
  private final String name;
  private final Set<String> cacheNames;
  private final String key;
  private final String keyGenerator;
  private final String cacheManager;
  private final String cacheResolver;
  private final String condition;
  private final String toString;
  // BasicOperation规定的要实现的getCacheNames方法就很简单了
  @Override
  public Set<String> getCacheNames() {
    return this.cacheNames;
  }
  // 比较奇怪的是，它的构造函数传入了一个Builder对象，然后这个Builder的抽象类写在这里
  // CacheEvictOperation、CachePutOperation等继承了这个Builder
  protected CacheOperation(Builder b) {
    // 这是构造函数
		this.name = b.name;
		this.cacheNames = b.cacheNames;
		this.key = b.key;
		this.keyGenerator = b.keyGenerator;
		this.cacheManager = b.cacheManager;
		this.cacheResolver = b.cacheResolver;
		this.condition = b.condition;
		this.toString = b.getOperationDescription().toString();
	}
  // 这是那个内部抽象类，这里的实现没有什么特殊的，就是给了一些默认值而已
  // 而且在一些set方法里，比如setCondition规定了condition不能为null，否则报错
  // 不过这些规定不能为null的细节就不在本文中讨论了
  public abstract static class Builder {
		private String name = "";
		private Set<String> cacheNames = Collections.emptySet();
		private String key = "";
		private String keyGenerator = "";
		private String cacheManager = "";
		private String cacheResolver = "";
		private String condition = "";
    // 规定了一个用来build CacheOperation的抽象方法
    public abstract CacheOperation build();
  }
}
```



然后我们看到，有三个类继承了`CacheOperation`这个抽象类，正好对应`@CachePut`、`CacheEvict`和`@Cacheable`这三个表示缓存操作的注解。

## `CachePutOperation`

`@CachePut`注解多了一个`unless`属性，所以这里的`CachePutOperation`里的field也多一个：

```java
public CachePutOperation extends CacheOperation {
  @Nullable
  private final String unless;
  // 然后它的构造函数也是使用builder
  public CachePutOperation(CacheOperation.Builder b) {
    super(b);
    this.unless = b.unless;
  }
  // 然后在这里继承一下CacheOperation.Builder
  public static class Builder extends CacheOperation.Builder {
    @Nullable
		private String unless;
		public CachePutOperation build() {
			return new CachePutOperation(this);
		}
  }
}
```



`CacheEvictOperation`、`CacheableOperation`这两个类和上面的`CachePutOperation`是差不多的套路，这里就不贴代码了，都比较简单。

# `JCacheOperation`

反倒是为了支持JSR-107，导致Spring Cache的代码开始变的复杂。
`JCacheOperation`的源码如下：

```java
// CacheMethodDetails是来自于JSR-107的一个接口，用来表示被JSR-107所提供的一些注解能够带有的静态信息
// 这些注解包括@CacheResult、@CachePut、@CacheRemove、@CacheRemoveAll
public interface JCacheOperation<A extends Annotation> extends BasicOperation, CacheMethodDetails<A> {
  // 这里的CacheResolver就是Spring Cache里的CacheResolver
  CacheResolver getCacheResolver();
  // CacheInvocationParameter是来自于JSR-107的一个接口
  // 它用来表示要给被intercepted(拦截)的方法调用的参数，包含了这个参数的value以及相关参数的静态类型和被注解修饰所带有的注释信息
  // getAllParameters这个方法就是根据输入的方法参数返回CacheInvocationParameter的实例
  CacheInvocationParameter[] getAllParameters(Object... values);
}
```



## `AbstractJCacheOperation`

然后来看一个`JCacheOperation`的简单实现，抽象类`AbstractJCacheOperation`：

```java
public abstract class AbstractJCacheOperation<A extends Annotation> implements JCacheOperation<A> {
  private final CacheMethodDetails<A> methodDetails;
  private final CacheResolver cacheResolver;
  // 这个是用来表示被包装的方法所带有的参数的信息的
  // 而这个方法实际上就是methodDetails里所封装对象
  // 所以在构造函数中是做了一些处理的，具体的看后面的构造函数
  protected final List<CacheParameterDetail> allParameterDetails;
  protected AbstractJCacheOperation(CacheMethodDetails<A> methodDetails, CacheResolver cacheResolver) {
		Assert.notNull(methodDetails, "method details must not be null.");
		Assert.notNull(cacheResolver, "cache resolver must not be null.");
		this.methodDetails = methodDetails;
		this.cacheResolver = cacheResolver;
		this.allParameterDetails = initializeAllParameterDetails(methodDetails.getMethod());
	}
  // 这个初始化的方法就是把methodDetails中封装的方法参数取出来，封装成List<CacheParameterDetail>
  private static List<CacheParameterDetail> initializeAllParameterDetails(Method method) {
		List<CacheParameterDetail> result = new ArrayList<>();
		for (int i = 0; i < method.getParameterCount(); i++) {
			CacheParameterDetail detail = new CacheParameterDetail(method, i);
			result.add(detail);
		}
		return result;
	}
  // 那就先来看一下这个CacheParameterDetail是怎么样的一个类了
  protected static class CacheParameterDetail {
    private final Class<?> rawType;
    private final Set<Annotation> annotations;
    private final int parameterPosition;
    private final boolean isKey;
    private final boolean isValue;
    // 构造函数
    public CacheParameterDetail(Method method, int parameterPosition) {
      this.rawType = method.getParameterTypes()[parameterPosition];
      this.annotations = new LinkedHashSet<>();
      boolean foundKeyAnnotation = false;
      boolean foundValueAnnotation = false;
      for (Annotation annotation : method.getParameterAnnotations()[parameterPosition]) {
        this.annotations.add(annotation);
        // 这两个地方调用了JSR-107的api来判断是否是key或者value的参数
        if (CacheKey.class.isAssignableFrom(annotation.annotationType())) {
          foundKeyAnnotation = true;
        }
        if (CacheValue.class.isAssignableFrom(annotation.annotationType())) {
          foundValueAnnotation = true;
        }
      }
      this.parameterPosition = parameterPosition;
      this.isKey = foundKeyAnnotation;
      this.isValue = foundValueAnnotation;
    }
  }
  // 这个方法返回一个ExceptionTypeFilter用于过滤调用方法时抛出的异常
  public abstract ExceptionTypeFilter getExceptionTypeFilter();
}
```



然后是对`AbstractJCacheOperation`这个类的继承，它的子类有`CacheRemoveAllOperation`和`AbstractJCacheKeyOperation`两个。可以看到它这里把来自于JSR-107的缓存操作分为了两类：

- 不需要key的操作，`@CacheRemoveAll`
- 需要key的操作，`@CacheResult`、`@CachePut`、`@CacheRemove`

### `CacheRemoveAllOperation`

对于不需要key的操作来说，`CacheRemoveAllOperation`的实现就比较简单了：

```java
// 这里的CacheRemoveAll是来自于JSR-107的注解 @CacheRemoveAll
public CacheRemoveAllOperation extends AbstractJCacheOperation<CacheRemoveAll> {
  private final ExceptionTypeFilter exceptionTypeFilter;
  @Override
  public ExceptionTypeFilter getExceptionTypeFilter() {
    return this.exceptionTypeFilter;
  }
}
```



### `AbstractJCacheKeyOperation`

对于需要key的操作，又出了一个抽象类：

```java
public abstract class AbstractJCacheKeyOperation<A extends Annotation> extends AbstractJCacheOperation<A> {
  private final KeyGenerator keyGenerator;
  // 这个就是用来报错作为key的参数的列表了
  private final List<CacheParameterDetail> keyParameterDetails;
  // 所以套路还是一样的，在构造函数里有一个对 keyParameterDetails 进行初始化的操作
  protected AbstractJCacheKeyOperation(CacheMethodDetails<A> methodDetails,
    CacheResolver cacheResolver, KeyGenerator keyGenerator) {
      super(methodDetails, cacheResolver);
      this.keyGenerator = keyGenerator;
      this.keyParameterDetails = initializeKeyParameterDetails(this.allParameterDetails);
  }
  // 初始化的过程就是从所有的Parameter里取出KeyParameter相关数据的过程
  private static List<CacheParameterDetail> initializeKeyParameterDetails(List<CacheParameterDetail> allParameters) {
    List<CacheParameterDetail> all = new ArrayList<>();
    List<CacheParameterDetail> annotated = new ArrayList<>();
    for (CacheParameterDetail allParameter : allParameters) {
      if (!allParameter.isValue()) {
        all.add(allParameter);
      }
      if (allParameter.isKey()) {
        annotated.add(allParameter);
      }
    }
    // 这里有点意思了，如果没有对入参标记哪个为 key 的话，就把所有的入参都标记为 key
    // 正好对应了某个默认的生成key的逻辑：如果没有标明哪个parameter用来生成key并且没有指定keyGenerator的话，就使用所有的入参进行一次哈希计算然后生成key
    return (annotated.isEmpty() ? all : annotated);
  }
}
```



剩下的这三个依赖于key的缓存操作 `CacheResultOperation`、`CacheRemoveOperation`、`CachePutOperation`，其实都是差不多的，所以这里就不分析了。



## SpringBoot配置Cache

项目里面要增加一个应用缓存，原本想着要怎么怎么来整合ehcache和springboot，做好准备配置这个配置那个，结果只需要做三件事：

- pom依赖
- 写好一个ehcache的配置文件
- 在boot的application上加上注解@EnableCaching. 
  这就完事了，是不是很魔幻。

**pom依赖**

```xml
<dependency>
            <groupId>net.sf.ehcache</groupId>
            <artifactId>ehcache</artifactId>
            <version>2.10.5</version>
</dependency>
```

**配置文件**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache>
    <!-- 设定缓存的默认数据过期策略 -->
    <defaultCache
            maxElementsInMemory="500"
            maxElementsOnDisk="2000"
            eternal="false"
            overflowToDisk="true"
            timeToIdleSeconds="90"
            timeToLiveSeconds="300"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="300"/>
</ehcache>
```

**应用上加上EnableCaching注解**

```java
@SpringBootApplication
@EnableCaching
public class EhCacheApplication {

    public static void main(String[] args) {
        SpringApplication.run(EhCacheApplication.class, args);
    }
}
```

然后就可以在代码里面使用cache注解了，像这样。

```java
@CachePut(value = "fish-ehcache", key = "#person.id")
    public Person save(Person person) {
        System.out.println("为id、key为:" + person.getId() + "数据做了缓存");
        return person;
    }

    @CacheEvict(value = "fish-ehcache")
    public void remove(Long id) {
        System.out.println("删除了id、key为" + id + "的数据缓存");
    }


    @Cacheable(value = "fish-ehcache", key = "#person.id")
    public Person findOne(Person person) {
        findCount.incrementAndGet();
        System.out.println("为id、key为:" + person.getId() + "数据做了缓存");
        return person;
    }
```

很方便对不对。下面，我们就来挖一挖，看看spring是怎么来做到的。主要分成两部分，一是启动的时候做了什么，二是运行的时候做了什么，三是和第三方缓存组件的适配



### 启动的时候做了什么

这个得从@EnableCaching标签开始，在使用缓存功能时，在springboot的Application启动类上需要添加注解@EnableCaching，这个标签引入了

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({CachingConfigurationSelector.class})
public @interface EnableCaching {
    boolean proxyTargetClass() default false;
    AdviceMode mode() default AdviceMode.PROXY;
    int order() default 2147483647;
}
```



引入了CachingConfigurationSelector类，这个类便开启了缓存功能的配置。这个类添加了AutoProxyRegistrar.java,ProxyCachingConfiguration.java两个类。

- AutoProxyRegistrar : 实现了ImportBeanDefinitionRegistrar接口。这里看不懂，还需要继续学习。
- ProxyCachingConfiguration : 是一个配置类，生成了BeanFactoryCacheOperationSourceAdvisor，CacheOperationSource，和CacheInterceptor这三个bean。

CacheOperationSource封装了cache方法签名注解的解析工作，形成CacheOperation的集合。CacheInterceptor使用该集合过滤执行缓存处理。解析缓存注解的类是SpringCacheAnnotationParser，其主要方法如下

```java
/**
由CacheOperationSourcePointcut作为注解切面，会解析
SpringCacheAnnotationParser.java
扫描方法签名，解析被缓存注解修饰的方法，将生成一个CacheOperation的子类并将其保存到一个数组中去
**/
protected Collection<CacheOperation> parseCacheAnnotations(SpringCacheAnnotationParser.DefaultCacheConfig cachingConfig, AnnotatedElement ae) {
        Collection<CacheOperation> ops = null;
        //找@cacheable注解方法
        Collection<Cacheable> cacheables = AnnotatedElementUtils.getAllMergedAnnotations(ae, Cacheable.class);
        if (!cacheables.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var5 = cacheables.iterator();

            while(var5.hasNext()) {
                Cacheable cacheable = (Cacheable)var5.next();
                ops.add(this.parseCacheableAnnotation(ae, cachingConfig, cacheable));
            }
        }
        //找@cacheEvict注解的方法
        Collection<CacheEvict> evicts = AnnotatedElementUtils.getAllMergedAnnotations(ae, CacheEvict.class);
        if (!evicts.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var12 = evicts.iterator();

            while(var12.hasNext()) {
                CacheEvict evict = (CacheEvict)var12.next();
                ops.add(this.parseEvictAnnotation(ae, cachingConfig, evict));
            }
        }
        //找@cachePut注解的方法
        Collection<CachePut> puts = AnnotatedElementUtils.getAllMergedAnnotations(ae, CachePut.class);
        if (!puts.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var14 = puts.iterator();

            while(var14.hasNext()) {
                CachePut put = (CachePut)var14.next();
                ops.add(this.parsePutAnnotation(ae, cachingConfig, put));
            }
        }
        Collection<Caching> cachings = AnnotatedElementUtils.getAllMergedAnnotations(ae, Caching.class);
        if (!cachings.isEmpty()) {
            ops = this.lazyInit(ops);
            Iterator var16 = cachings.iterator();

            while(var16.hasNext()) {
                Caching caching = (Caching)var16.next();
                Collection<CacheOperation> cachingOps = this.parseCachingAnnotation(ae, cachingConfig, caching);
                if (cachingOps != null) {
                    ops.addAll(cachingOps);
                }
            }
        }
        return ops;
}
```

解析Cachable,Caching,CachePut,CachEevict 这四个注解对应的方法都保存到了Collection<CacheOperation> 集合中。

### 执行方法时做了什么

执行的时候，主要使用了CacheInterceptor类。

```java
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {
    public CacheInterceptor() {
    }

    public Object invoke(final MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        CacheOperationInvoker aopAllianceInvoker = new CacheOperationInvoker() {
            public Object invoke() {
                try {
                    return invocation.proceed();
                } catch (Throwable var2) {
                    throw new ThrowableWrapper(var2);
                }
            }
        };

        try {
            return this.execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
        } catch (ThrowableWrapper var5) {
            throw var5.getOriginal();
        }
    }
}
```

这个拦截器继承了CacheAspectSupport类和MethodInterceptor接口。其中CacheAspectSupport封装了主要的逻辑。比如下面这段。



```java
/**
CacheAspectSupport.java
执行@CachaEvict @CachePut @Cacheable的主要逻辑代码
**/

private Object execute(final CacheOperationInvoker invoker, Method method, CacheAspectSupport.CacheOperationContexts contexts) {
        if (contexts.isSynchronized()) {
            CacheAspectSupport.CacheOperationContext context = (CacheAspectSupport.CacheOperationContext)contexts.get(CacheableOperation.class).iterator().next();
            if (this.isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
                Object key = this.generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
                Cache cache = (Cache)context.getCaches().iterator().next();

                try {
                    return this.wrapCacheValue(method, cache.get(key, new Callable<Object>() {
                        public Object call() throws Exception {
                            return CacheAspectSupport.this.unwrapReturnValue(CacheAspectSupport.this.invokeOperation(invoker));
                        }
                    }));
                } catch (ValueRetrievalException var10) {
                    throw (ThrowableWrapper)var10.getCause();
                }
            } else {
                return this.invokeOperation(invoker);
            }
        } else {
            /**
            执行@CacheEvict的逻辑，这里是当beforeInvocation为true时清缓存
            **/
            this.processCacheEvicts(contexts.get(CacheEvictOperation.class), true, CacheOperationExpressionEvaluator.NO_RESULT);
            //获取命中的缓存对象
            ValueWrapper cacheHit = this.findCachedItem(contexts.get(CacheableOperation.class));
            List<CacheAspectSupport.CachePutRequest> cachePutRequests = new LinkedList();
            if (cacheHit == null) {
                //如果没有命中，则生成一个put的请求
                this.collectPutRequests(contexts.get(CacheableOperation.class), CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
            }


            Object cacheValue;
            Object returnValue;
            /**
                如果没有获得缓存对象，则调用业务方法获得返回对象，hasCachePut会检查exclude的情况
            **/
            if (cacheHit != null && cachePutRequests.isEmpty() && !this.hasCachePut(contexts)) {
                cacheValue = cacheHit.get();
                returnValue = this.wrapCacheValue(method, cacheValue);
            } else {
                
                returnValue = this.invokeOperation(invoker);
                cacheValue = this.unwrapReturnValue(returnValue);
            }

            this.collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);
            Iterator var8 = cachePutRequests.iterator();

            while(var8.hasNext()) {
                CacheAspectSupport.CachePutRequest cachePutRequest = (CacheAspectSupport.CachePutRequest)var8.next();
                /**
                执行cachePut请求，将返回对象放到缓存中
                **/
                cachePutRequest.apply(cacheValue);
            }
            /**
            执行@CacheEvict的逻辑，这里是当beforeInvocation为false时清缓存
            **/
            this.processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);
            return returnValue;
        }
    }
```

上面的代码片段比较核心，均是cache的内容，对于aop的源码，这里不详细展开，应该单起一篇文章进行研究。主要的类和接口都在spring的context中，org.springframework.cache包中。

### 和第三方缓存组件的适配

通过以上的分析，知道了spring cache功能的来龙去脉，下面需要分析的是，为什么只需要maven声明一下依赖，spring boot 就可以自动就适配了.

在上面的执行方法中，我们看到了**cachePutRequest.apply(cacheValue)** ,这里会操作缓存，CachePutRequest是CacheAspectSupport的内部类。



```java
private class CachePutRequest {
        private final CacheAspectSupport.CacheOperationContext context;
        private final Object key;
        public CachePutRequest(CacheAspectSupport.CacheOperationContext context, Object key) {
            this.context = context;
            this.key = key;
        }
        public void apply(Object result) {
            if (this.context.canPutToCache(result)) {
                //从context中获取cache实例，然后执行放入缓存的操作
                Iterator var2 = this.context.getCaches().iterator();
                while(var2.hasNext()) {
                    Cache cache = (Cache)var2.next();
                    CacheAspectSupport.this.doPut(cache, this.key, result);
                }
            }
        }
    }
```

Cache是一个标准接口，其中EhCacheCache就是EhCache的实现类。这里就是SpringBoot和Ehcache之间关联的部分，那么context中的cache列表是什么时候生成的呢。答案是CacheAspectSupport的getCaches方法

```java
protected Collection<? extends Cache> getCaches(CacheOperationInvocationContext<CacheOperation> context, CacheResolver cacheResolver) {
        Collection<? extends Cache> caches = cacheResolver.resolveCaches(context);
        if (caches.isEmpty()) {
            throw new IllegalStateException("No cache could be resolved for '" + context.getOperation() + "' using resolver '" + cacheResolver + "'. At least one cache should be provided per cache operation.");
        } else {
            return caches;
        }
    }
```

而获取cache是在每一次进行进行缓存操作的时候执行。可以看一下调用栈

![图片描述](https://segmentfault.com/img/bVbjwkL?w=746&h=291)

貌似有点跑题，拉回来... 在spring-boot-autoconfigure包里，有所有自动装配相关的类。这里有个EhcacheCacheConfiguration类 ，如下

```java
@Configuration
@ConditionalOnClass({Cache.class, EhCacheCacheManager.class})
@ConditionalOnMissingBean({CacheManager.class})
@Conditional({CacheCondition.class, EhCacheCacheConfiguration.ConfigAvailableCondition.class})
class EhCacheCacheConfiguration {
 ......
 static class ConfigAvailableCondition extends ResourceCondition {
        ConfigAvailableCondition() {
            super("EhCache", "spring.cache.ehcache", "config", new String[]{"classpath:/ehcache.xml"});
        }
    }    
}
```

这里会直接判断类路径下是否有ehcache.xml文件

