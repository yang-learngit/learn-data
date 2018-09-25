# 目的

​	熟悉Redis常用的数据结构格式及其对应的指令使用。
# 前言

​	本文主要介绍了Redis的string，hash，list，set，zset等五种常见的数据结构，分别介绍了数据结构的格式及对应常用的api使用，以及各数据结构在不同情形下Redis内部结构的选择，最后简单的阐述各种数据结构在实际应用中一些使用场景。


# 正文


## 字符串(string)

​	字符串类型是Redis最基础的数据结构。在Redis中，所有的键都是字符串类型，而且其它几种数据结构都是在字符串的基础上构建的，字符串类型的值实际可以是字符串（简单字符串、复杂字符串（例如JSON、XML））、数字（整数、浮点数），甚至是二进制（图片、音频），但是要注意的是**值最大不能超过512M**。

### 常用命令

#### 设置值 	
```
指令： set key value [ex seconds] [px milliseconds] [nx|xx]
举例： set hello world；//设置键为hello，值为world的键值对，返回ok代表设置成功
其中set指令有几个选项，下面分别进行说明：
1. ex seconds：为键设置秒级过期时间
2. px milliseconds：为键设置毫秒级过期时间
3. nx：键必须不存在，才可以设置成功，用于添加
4. xx：与nx相反，键必须存在，才可以设置成功，用于更新
```

 除了set命令，Redis还为我们提供了setex和setnx命令，其作用是与上面的ex跟nx一样，命令格式如下：

```
setex key seconds value
setnx key value
```
####  获取值

```
指令： get key
举例： get hello；//获取key为hello的值，若key存在，返回相应的值，否则返回nil
```
#### 批量设置值

```
指令： mset key value [key value ...]
举例： mset name1 boy name2 girl name3 unknow；//批量设置key-value，返回ok代表设置成功
```
#### 批量获取值

```
指令： mget key [key ...]
举例： mget name1 name2 name4；//批量获取key对应的值，如果当前key不存在，返回nil，结果按传入键顺序返回
```

Redis提供了批量操作命令可以有效提高开发效率，加入没有mget命令，获取n个key的值需要执行n次get命令，其中需要n次网络传输时间 + n次get命令执行时间，而有了mget命令，只需要1次网络传输时间 + n次get命令。但是要注意的是**每次批量获取的key数要适当**，不然由于Redis的单线程架构，键数太多容易造成Redis阻塞。
#### 计数

```
指令： incr key
举例： incr num；//对值做自增操作,返回自增后的值
其中incr命令分为以下几种情形：
1. 值不是整数，返回错误
2. 值是整数，返回自增后的结果
3. 键不存在，按照值为0自增，返回结果为1
```

除了incr命令，Redis还提供了decr（自减）、incrby（自增指定数字）、decrby（自减指定数字）、incrbyfloat（自增浮点数），命令格式如下：

```
decr key
incrby key increment
decrby key decrement
incrbyfloat key increment
```
### 内部编码

​	Redis字符串类型的内部编码有3种，分别是int、embstr、raw，Redis会根据当前值的类型和长度决定使用哪种内部编码实现，可以使用`object encoding key`指令进行验证，这几种内部编码的使用情况分别如下：

	1. int：8个字节的长整型
	2. embstr：小于等于39个字节的字符串
	3. raw：大于39个字节的字符串
### 使用场景

#### 缓存信息载体

​	上面介绍时说过，Redis字符串的值可以为简单字符串，也可以为复杂字符串（JSON，XML），甚至是Binary，这也意味着我们可以将Java对象序列为Json或Binary后进行保存。

​	在这里，了解过Redis的hash数据结构的童鞋应该知道，hash也常被用于缓存信息载体，那么在这里到底是要采用string还是hash进行保存信息载体呢？

个人认为这取决于你对数据访问的访问方式，分为以下两种情况进行讨论：
1.string：
（1）在多数情况下，你需要用到的对象字段较多，可以考虑采用string；
2.hash：
（1）在多数情况下，你需要用到的对象字段较少，可以考虑采用hash；
（2）在多数情况下，总是知道哪些字段是要用的，可以考虑采用hash；

#### 用于计数器

​	如常见的视频网站播放量，论坛帖子的点击量，点赞人数等等。

#### 共享session

​	在网站的集群加负载均衡场景下，可用于做session服务器，统一管理session。

#### 用于操作限制

​	用于对网站接口的访问次数限制，如网站的短信接口不被频繁访问，会限制用户每分钟获取验证码的频率，例如一分钟不能超过5次。



## 哈希(hash)

​	哈希类型（hash）是指键值本身又是一个键值对结构，在哈希类型中，映射关系叫作field-value，注意这里的value是指field对应的值，不是键对应的值。

### 常用命令

####  设置值

``` 
指令： hset key field value
示例： hset user:1 name tony；//将哈希表Key为user:1中的域name的值设为tony
说明： 
1.将哈希表Key中的域field的值设为value 
2.key不存在，一个新的Hash表被创建
3.field已经存在，旧的值被覆盖

Redis还提供了hsetnx命令，，它们的关系就像set和setnx命令一样，只不过作用域由键变为field
```
####  获取值

``` 
指令： hget key field
示例： hget user:1 name；//返回tony
说明：
1.返回哈希表key中给定域field的值
2.如果键或field不存在，会返回nil
```
####  删除field

```
指令： hdel key field [field ...]
示例： hdel user:1 name；//删除key为user:1 filed为name的值
说明： hdel会删除一个或多个field，返回结果为成功删除field的个数
```
####  计算field个数

```
指令： hlen key
示例： hlen user:1；//返回1
```
####  批量设置或获取field-value

```
//批量设置field-value
指令： hmset key field value [field value ...]
示例： hmset user:1 name tony age 28

//批量获取filed-value
指令： hmget key field [field ...]
示例： hmget user:1 name age；//返回tony，28
```
####  判断field是否存在

```
指令： hexists key field
示例： hexists user:1 name；//如果field存在，返回结果为1，否则返回结果0
```
####  获取所有field

```
指令： hkeys key
示例： hkeys user:1；//返回name，age
```
####  获取所有value

```
指令： hvals key
示例： hvals user:1；//返回tony，28
```
####  获取所有的field-value

```
指令： hgetall key
示例： hgetall user:1；//返回name tony age 28
说明： 在使用hgetall命令时，如果当前hash集合的个数比较多，有可能会造成Redis阻塞，如果只是获取部分filed-value，建议使用hmget，如果一定要获取全部field-value，建议使用hscan命令，该命令会渐进式遍历哈希类型。
```
#### 计数

```
指令： hincrby key field increment
示例： hincrby user:1 age 8；//为哈希表user:1中的域age加上增量8

指令： hincrbyfloat key field increment
示例： hincrbyfloat user:1 age 10.8；//为哈希表user:1中的域age加上增量10.8
```
### 内部编码

​	哈希类型的内部编码有两种，分别是ziplist（压缩列表）和 hashtable（哈希表）。

​	ziplist（压缩列表）：当哈希类型元素个数小于hash-max-ziplist-entries配置（默认512个）、同时所有值都小于hash-max-ziplist-value配置（默认64字节）时，Redis会使用ziplist作为哈希的内部实现，ziplist使用更加紧凑的结构实现多个元素的连续存储，所以在节省内存方面比hashtable更加优秀。

​	hashtable（哈希表）：当哈希类型无法满足ziplist的条件时，Redis会使用hashtable作为哈希的内部实现，因为此时ziplist的读写效率会下降，而hashtable的读写时间复杂度为O（1）。

### 使用场景

#### 缓存信息载体

​	Redis的哈希类型也常被用与缓存对象信息，相比使用字符串序列化缓存用户信息，哈希类型更直观，且在更新操作上会更加方便（省去序列化与反序列化造成的开销）。




## 列表(list)

​	列表（list）类型是用来存储多个有序的字符串，列表中的每个字符串称为元素（element），一个列表最多可以存储232-1个元素。在Redis中，列表类型有两个特点，第一为列表中的元素是有序的，第二是列表中的元素是可以重复的，基于以上两个特点，我们可以对列表两端进行插入（push）和弹出（pop），还可以获取指定范围的元素列表、获取指定索引下标的元素等。总而言之，列表是一种比较灵活的数据结构，它可以充当栈和队列的角色，在实际开发上有很多应用场景。

### 常用命令

####  添加元素

```
//从右边插入元素
指令： rpush key value [value...]
举例： rpush names tony mike jaxi；//从右边插入key为names，value为tony mike jaxi的元素

//从左边插入元素
指令： lpush key value [value...]
举例： lpush names tony mike jaxi；//从左边插入key为names，value为tony mike jaxi的元素

//向某个元素前或者后插入元素
指令： linsert key before|after pivot value
举例： linsert names before mike nanxi；//从列表中找到value为mike的值，在前面插入value为nanxi的元素
```

#### 查找元素

```
//获取指定范围内的元素列表
指令： lrange key start end
举例： lrange names 1 3；// lrange的end下标是有包含了自身，如该示例是获取第2到第4个元素
关于lrange指令的几点操作说明：
1. lrange的end下标是有包含了自身元素
2. 索引下标从左到右分别是0到N-1，从右到左分别是-1到-N

//获取列表指定索引下标的元素
指令： lindex key index
举例： lindex names -1；//获取当前列表最后一个元素为‘tony’

//获取列表长度
指令： llen key
举例： llen names；//返回key为names的列表长度为4
```
#### 删除元素

```
//从列表左侧删除元素
指令： lpop key
举例： lpop names；//从左边删除列表names，如上示例删除的元素为‘jaxi’

//从列表右侧删除元素
指令： rpop key
举例： rpop names；//从右边删除列表names，如上示例删除的元素为‘tony’

//删除指定元素
指令： lrem key count value
举例： lrem names 1 nanxi；//删除元素为‘nanxi’
lrem命令会从列表中找到等于value的元素进行删除，根据count的不同分为三种情况：
1. count>0，从左到右，删除最多count个元素
2. count<0，从右到左，删除最多count绝对值个元素
3. count=0，删除所有元素

//按照索引范围修剪列表
指令： ltrim key start end
举例： ltrim names 1 3；//保留names列表的第二到第四个元素
```
#### 修改元素

```
//修改指定索引下标的元素
指令： lset key index newValue
举例： lset names 0 houyi；//将names列表的第一个元素修改为houyi，下标规则同上
```
####  阻塞式命令

```
Redis为list列表类型提供了brpop与blpop两个阻塞式命令，blpop和brpop是lpop和rpop的阻塞版本，它们除了弹出方向不同，使用方法基本相同。

指令：blpop key [key ...] timeout 或 brpop key [key ...] timeout
其中key[key...]为多个列表的键，timeout为阻塞时间（单位：秒）

关于blpop和brpop的使用说明，这里分为两种情况：
第一，列表为空：如果timeout=3，那么客户端要等到3秒后返回，如果timeout=0，那么客户端一直阻塞等下去；
第二，列表不为空：客户端会立即返回

在使用brpop时，有两点要注意：
第一点，如果是多个键，那么brpop会从左至右遍历键，一旦有一个键能弹出元素，客户端立即返回；
第二点，如果多个客户端对同一个键执行brpop，那么最先执行brpop命令的客户端可以获取到弹出的值；
```
### 内部编码

​	Redis列表类型的内部编码有两种：
​	1.ziplist（压缩列表）：当列表的元素个数小于list-max-ziplist-entries配置（默认512个），同时列表中每个元素的值都小于list-max-ziplist-value配置时（默认64字节），Redis会选用ziplist来作为列表的内部实现来减少内存的使用；
​	2.linkedlist（链表）：当列表类型无法满足ziplist的条件时，Redis会使用linkedlist作为列表的内部实现；
​	3.Redis3.2版本提供了quicklist内部编码，简单地说它是以一个ziplist为节点的linkedlist，它结合了ziplist和linkedlist两者的优势，为列表类型提供了一种更为优秀的内部编码实现；
### 使用场景

#### 消息队列

​	Redis提供的lpush+brpop或rpush+blpop可以实现延迟队列。

​	开发口诀：

​	lpush+lpop=Stack（栈）
​	lpush+rpop=Queue（队列）
​	lpsh+ltrim=Capped Collection（有限集合）
	lpush+brpop=Message Queue（消息队列）



## 集合(set)

​	集合（set）类型也是用来保存多个的字符串元素，但和列表类型不一样的是，集合中不允许有重复元素，并且集合中的元素是无序的，不能通过索引下标获取元素。Redis除了支持集合内的增删改查，同时还支持多个集合取交集、并集、差集，合理地使用好集合类型，能在实际开发中解决很多实际问题。

### 常用命令

####  集合内操作

##### 添加元素

```
指令： sadd key element [element ...]
示例： sadd myset java python；//向集合中添加元素java，python，返回结果为添加成功的元素个数
```
##### 删除元素

```
指令： srem key element [element ...]
示例： srem myset java；//删除集合中的元素java，返回结果为成功删除元素个数
```
##### 计算元素个数

```
指令： scard key
示例： scard myset；//返回集合myset中的元素个数
说明： scard的时间复杂度为O（1），它不会遍历集合所有元素，而是直接用Redis内部的变量
```
##### 判断元素是否在集合中

```
指令： sismember key element
示例： sismember myset python；//如果存在集合内返回1，反之返回0
```
##### 随机从集合返回指定个数元素

```
指令： srandmember key [count]
示例： srandmember myset 2；//从集合myset中随机返回两个元素
说明： [count]是可选参数，如果不写默认为1
```
##### 从集合随机弹出元素

```
指令： spop key
示例： spop myset；//从集合随机选择一个元素进行删除，返回结果为删除的元素
说明： srandmember和spop都是随机从集合选出元素，两者不同的是spop命令执行后，元素会从集合中删除，而srandmember不会
```
##### 获取所有元素

```
指令： smembers key
示例： smembers myset；//返回java，python，返回结果是无序的
说明： smembers属于比较重的命令，如果元素过多存在阻塞Redis的可能性，这时候可以使用sscan来完成
```
####  集合间操作

##### 多个集合的交集

```
指令： sinter key [key ...]
示例： sinter myset1 myset2；//求集合myset1与myset2的交集
```
##### 多个集合的并集

```
指令： suinon key [key ...]
示例： suinon myset1 myset2；//求集合myset1与myset2的并集
```
##### 多个集合的差集

```
指令： sdiff key [key ...]
示例： sdiff myset1 myset2；//返回一个集合的全部成员，该集合是第一个Key对应的集合和后面key对应的集合的差集
```
##### 将交集、并集、差集的结果保存

```
sinterstore destination key [key ...]
suionstore destination key [key ...]
sdiffstore destination key [key ...]
示例： sinterstore myset1_2 myset1 myset2；//将myset1和myset2两个集合的交集结果保存在myset1_2中，myset1_2本身也是集合类
说明：集合间的运算在元素较多的情况下会比较耗时，所以Redis提供了上面三个命令（原命令+store）将集合间交集、并集、差集的结果保存在destination key中
```
### 内部编码

​	集合类型的内部编码有两种，intset（整数集合）和 hashtable（哈希表）

​	intset（整数集合）：当集合中的元素都是整数且元素个数小于set-maxintset-entries配置（默认512个时，Redis会选用intset来作为集合的内部实现，从而减少内存的使用。

​	hashtable（哈希表）：当集合类型无法满足intset的条件时，Redis会使用hashtable作为集合的内部实现。
### 使用场景

#### 社交共同好友

​	比如在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合，利用 Redis 为集合提供的求交集、并集、差集等操作，那么就可以非常方便的实现如共同关注、共同喜好、二度好友等功能。



## 有序集合(zset)
​	有序集合（zset）与集合一样，元素都不能重复，但与集合不同的是，有序集合中的元素是有顺序的。与列表使用下标作为排序依据不同，有序集合为每个元素设置一个分数（score）作为排序的依据，在有序集合中，score是可以重复的。
### 常用命令

####  集合内操作

##### 添加元素

```
指令： zadd key score member [score member ...]
示例： zadd myzset 90 tony；//向有序集合myzset添加用户tony和他的分数90
说明： Redis在3.2版本为zadd命令增加了x、xx、ch、incr四个选项
1. nx：member必须不存在，才可以设置成功，用于添加
2. xx：member必须存在，才可以设置成功，用于更新
3. ch：返回此次操作后，有序集合元素和分数发生变化的个数
4. incr：对score做增加
```
##### 计算元素个数

```
指令： zcard key
示例： zcard myzset；//返回元素个数为1，与scard命令一样，zcard的时间复杂度为O（1）
```
##### 计算某个成员的分数

```
指令： zscore key member
示例： zscore myzset tony；//返回tony的分数90，如果成员不存在，返回nil
```
##### 计算成员的排名

```
指令：
zrank key member
zrevrank key member
说明：zrank是从分数从低到高返回排名，zrevrank反之，排名从0开始计算
```
##### 删除成员

```
指令： zrem key member [member ...]
示例： zrem myzset tony；//删除集合成员tony，返回删除成功个数
```
##### 增加成员的分数

```
指令： zincrby key increment member
示例： zincrby myzset 10 tony；//给集合成员的分数增加10分，变成100
```
##### 返回指定排名范围的成员

```
指令：
zrange key start end [withscores]
zrevrange key start end [withscores]
说明：有序集合是按照分值排名的，zrange是从低到高返回，zrevrange反之。果加上withscores选项，同时会返回成员的分数
```
##### 返回指定分数范围的成员

```
指令：
zrangebyscore key min max [withscores] [limit offset count]
zrevrangebyscore key max min [withscores] [limit offset count]
说明：zrangebyscore按照分数从低到高返回，zrevrangebyscore反之。withscores选项会同时返回每个成员的分数。[limit offset count]选项可以限制输出的起始位置和个数。
```
##### 返回指定分数范围成员个数

```
指令： zcount key min max
示例： zcount myzset 100 150；//返回100到150分的成员的个数
```
##### 删除指定排名内的升序元素

```
指令： zremrangebyrank key start end
示例： zremrangebyrank myzset 0 2；//删除myset第0到2排名的成员
```
##### 删除指定分数范围的成员

```
指令： zremrangebyscore key min max
示例： zremrangebyscore myzset (250 +inf；//将250分以上的成员全部删除
说明： min和max还支持开区间（小括号）和闭区间（中括号），-inf和+inf分别代表无限小和无限大
```
####  集合间操作
##### 多个集合的交集

```
zinterstore destination numkeys key [key ...] [weights weight [weight ...]] [aggregate sum|min|max]
```
##### 多个集合的并集

```
zunionstore destination numkeys key [key ...] [weights weight [weight ...]] [aggregate sum|min|max]
```
### 内部编码

​	有序集合类型的内部编码有两种，ziplist（压缩列表）和 skiplist（跳跃表）

​	ziplist（压缩列表）：当有序集合的元素个数小于zset-max-ziplistentries配置（默认128个），同时每个元素的值都小于zset-max-ziplist-value配置（默认64字节）时，Redis会用ziplist来作为有序集合的内部实现，ziplist可以有效减少内存的使用。

​	skiplist（跳跃表）：当ziplist条件不满足时，有序集合会使用skiplist作为内部实现，因为此时ziplist的读写效率会下降。
### 使用场景
#### 排行榜系统

​	视频网站需要对用户上传的视频做排行榜，榜单的维度可能是多个方面的：按照时间、按照播放数量、按照获得的赞数。