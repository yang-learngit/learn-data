# Redis持久化



## 1.目的

深入介绍Redis持久化实现原理与工作流程，对比Redis两种持久化方式的优缺点，讨论RDB与AOF持久化策略选择问题



## 2.前言

Redis是一种内存型数据库，一旦服务器进程退出，数据库的数据就会丢失。为了使Redis在重启后仍能保证数据不丢失,需要将数据从内存中以某种形式持久化到硬盘中。Redis支持两种持久化方式，一种是RDB方式，一种是AOF方式。



## 3.RDB

把当前进程数据生成快照保存到硬盘的过程， 触发RDB持久化过程分为手动触发和自动触发。

### 3.1触发机制

1、手动触发分别对应save和bgsave命令。

- **save**：会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在Redis服务器阻塞期间，服务器不能处理任何命令请求。**（线上环境要杜绝save的使用）**

- **bgsave**：会创建一个子进程，由子进程来负责创建RDB文件，父进程(即Redis主进程)则继续处理请求。



2、自动触发RDB的持久化机制。

- **save m n**：指定当m秒内发生n次变化时，会触发bgsave

- 在主从复制场景下，如果从节点执行全量复制操作，则主节点会执行bgsave命令，并将rdb文件发送给从节点

- 执行debug reload命令重新加载Redis时， 也会自动触发save操作
- 执行shutdown命令时，如果没有开启AOF持久化功能则自动执行rdb持久化

### 3.2执行流程

![bgsave的执行流程](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/bgsave%E7%9A%84%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

上图是bgsave的执行流程：

1)  Redis父进程首先判断：当前是否在执行save，或bgsave/bgrewriteaof的子进程，如果在执行则bgsave命令直接返回。bgsave/bgrewriteaof 的子进程不能同时执行，主要是基于性能方面的考虑：两个并发的子进程同时执行大量的磁盘写操作，可能引起严重的性能问题

2)  父进程执行fork操作创建子进程，这个过程中父进程是阻塞的，Redis不能执行来自客户端的任何命令

3)  父进程fork后，bgsave命令返回”Background saving started”信息并不再阻塞父进程，并可以响应其他命令

4)  子进程创建RDB文件，根据父进程内存快照生成临时快照文件，完成后对原有文件进行原子替换

5)  子进程发送信号给父进程表示完成，父进程更新统计信息

### 3.3RDB文件

- **存储路径**

RDB文件保存在dir配置指定的目录下， 文件名通过dbfilename配置指定。可以通过执行config set dir{newDir}和config set dbfilename{newFileName}运行期动态执行， 当下次运行时RDB文件会保存到新目录。

- **文件格式**

![RDB文件格式](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/RDB%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F.png)

各个字段的含义说明如下：

1)  REDIS：常量，保存着”REDIS”5个字符。

2)  db_version：RDB文件的版本号，注意不是Redis的版本号。

3)  SELECTDB 0 pairs：表示一个完整的数据库(0号数据库)，同理SELECTDB 3 pairs表示完整的3号数据库；只有当数据库中有键值对时，RDB文件中才会有该数据库的信息(上图所示的Redis中只有0号和3号数据库有键值对)；如果Redis中所有的数据库都没有键值对，则这一部分直接省略。其中：SELECTDB是一个常量，代表后面跟着的是数据库号码；0和3是数据库号码；pairs则存储了具体的键值对信息，包括key、value值，及其数据类型、内部编码、过期时间、压缩信息等等。

4)  EOF：常量，标志RDB文件正文内容结束。

5)  check_sum：前面所有内容的校验和；Redis在载入RBD文件时，会计算前面的校验和并与check_sum值比较，判断文件是否损坏。

- **压缩**

Redis默认采用LZF算法对RDB文件进行压缩。虽然压缩耗时，但是可以大大减小RDB文件的体积，因此压缩默认开启。可以通过参数config set rdbcompression{yes|no}动态修改。

### 3.4文件载入

RDB文件的载入工作是在服务器启动时自动执行的，并没有专门的命令。但是由于AOF的优先级更高，因此当AOF开启时，Redis会优先载入AOF文件来恢复数据；只有当AOF关闭时，才会在Redis服务器启动时检测RDB文件，并自动载入。服务器载入RDB文件期间处于阻塞状态，直到载入完成为止。**（在4.6章节再详细说明）**

### 3.5RDB常用配置

- **save m n**：bgsave自动触发的条件；如果没有save m n配置，相当于自动的RDB持久化关闭，不过此时仍可以通过其他方式触发
- **stop-writes-on-bgsave-error yes**：当bgsave出现错误时，Redis是否停止执行写命令；设置为yes，则当硬盘出现问题时，可以及时发现，避免数据的大量丢失；设置为no，则Redis无视bgsave的错误继续执行写命令，当对Redis服务器的系统(尤其是硬盘)使用了监控时，该选项考虑设置为no
- **rdbcompression yes**：是否开启RDB文件压缩
- **rdbchecksum yes**：是否开启RDB文件的校验，在写入文件和读取文件时都起作用；关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现
- **dbfilename dump.rdb**：RDB文件名
- **dir ./**：RDB文件和AOF文件所在目录

### 3.6RDB的优缺点

**优点：**

- RDB 是一个非常紧凑的文件，它保存了 Redis 在某个时间点上的数据集。 这种文件非常适合用于进行备份。
- RDB 非常适用于灾难恢复，它只有一个文件，并且内容都非常紧凑，可以（在加密后）将它传送到别的数据中心。
- RDB 可以最大化 Redis 的性能，父进程在保存 RDB 文件时唯一要做的就是 `fork` 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。
- RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

**缺点：**

- RDB不适合对数据持久化实时性要求比较高的情况。
  - 每次保存 RDB 的时候，Redis 都要 `fork()` 出一个子进程，并由子进程来进行实际的持久化工作。 在数据集比较庞大时， `fork()` 可能会非常耗时，造成服务器在某某毫秒内停止处理客户端。 虽然 AOF 重写也需要进行 `fork()` ，但无论 AOF 重写的执行间隔有多长，数据的耐久性都不会有任何损失。**（在下一章节讲解AOF时再具体讨论）**
- RDB文件使用特定二进制格式保存， Redis版本演进过程中有多个格式的RDB版本， 存在老版本Redis服务无法兼容新版RDB格式的问题。



## 4.AOF

以独立日志的方式记录每次写命令，重启时再重新执行AOF文件中的命令达到恢复数据的目的。


### 4.1开启AOF

Redis服务器默认开启RDB，关闭AOF；要开启AOF，需要在配置文件中配置：appendonly yes

### 4.2执行流程

![aof的执行流程](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/aof%E7%9A%84%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

上图是AOF的执行流程：

1)  命令追加(append)：将Redis的写命令追加到缓冲区aof_buf；

2)  文件写入(write)和文件同步(sync)：根据不同的同步策略将aof_buf中的内容同步到硬盘；

3)  文件重写(rewrite)：定期重写AOF文件，达到压缩的目的；

4)  当Redis服务器重启时， 可以加载AOF文件进行数据恢复。

### 4.3命令写入

AOF命令写入的内容直接是文本协议格式。 例如set hello world这条命令， 在AOF缓冲区会追加如下文本： 

`*3\r\n$3\r\nset\r\n$5\r\nhello\r\n$5\r\nworld\r\n `

1） AOF为什么直接采用文本协议格式？ 

- 文本协议具有很好的兼容性。 
- 开启AOF后， 所有写入命令都包含追加操作， 直接采用协议格式， 避免了二次处理开销。 
- 文本协议具有可读性， 方便直接修改和处理。 

2） AOF为什么把命令追加到aof_buf中？ 

Redis使用单线程响应命令， 如果每次写AOF文件命令都直接追加到硬盘， 那么性能完全取决于当前硬盘负载。先写入缓冲区aof_buf中， Redis可以提供多种缓冲区同步硬盘的策略，在性能和安全性方面做出平衡。 

### 4.4文件同步

Redis提供了多种AOF缓存区的同步文件策略，策略涉及到操作系统的write函数和fsync函数，说明如下：

为了提高文件写入效率，在现代操作系统中，当用户调用write函数将数据写入文件时，操作系统通常会将数据暂存到一个内存缓冲区里，当缓冲区被填满或超过了指定时限后，才真正将缓冲区的数据写入到硬盘里。这样的操作虽然提高了效率，但也带来了安全问题，如果计算机停机，内存缓冲区中的数据会丢失；因此系统同时提供了fsync、fdatasync等同步函数，可以强制操作系统立刻将缓冲区中的数据写入到硬盘里，从而确保数据的安全性。

AOF缓存区的同步文件策略由参数appendfsync控制，各个值的含义如下：

- always：命令写入aof_buf后立即调用系统fsync操作同步到AOF文件，fsync完成后线程返回。这种情况下，每次有写命令都要同步到AOF文件，硬盘IO成为性能瓶颈，Redis只能支持大约几百TPS写入，严重降低了Redis的性能；即便是使用固态硬盘，每秒大约也只能处理几万个命令，而且会大大降低SSD的寿命。
- no：命令写入aof_buf后调用系统write操作，不对AOF文件做fsync同步；同步由操作系统负责，通常同步周期为30秒。这种情况下，文件同步的时间不可控，而且缓冲区中堆积的数据会很多，数据安全性无法保证。
- everysec：命令写入aof_buf后调用系统write操作，write完成后线程返回；fsync同步文件操作由专门的线程每秒调用一次。**everysec是前述两种策略的折中，是性能和数据安全性的平衡，因此是Redis的默认配置，也是我们推荐的配置。**

### 4.5文件重写

随着时间流逝，Redis服务器执行的写命令越来越多，AOF文件也会越来越大；过大的AOF文件不仅会影响服务器的正常运行，也会导致数据恢复需要的时间过长。

**文件重写**是指定期重写AOF文件，减小AOF文件的体积。

**AOF重写**是把Redis进程内的数据转化为写命令，同步到新的AOF文件；不会对旧的AOF文件进行任何读取、写入操作!

对于AOF持久化来说，文件重写虽然是强烈推荐的，但并不是必须的；即使没有文件重写，数据也可以被持久化并在Redis启动的时候导入；因此在一些实现中，会关闭自动的文件重写，然后通过定时任务在每天的某一时刻定时执行。

文件重写之所以能够压缩AOF文件，原因在于：

- 过期的数据不再写入文件
- 无效的命令不再写入文件，如有些数据被重复设值(set mykey v1, set mykey v2)、有些数据被删除了(sadd myset v1, del myset)等等
- 多条命令可以合并为一个，如sadd myset v1, sadd myset v2, sadd myset v3可以合并为sadd myset v1 v2 v3。不过为了防止单条命令过大造成客户端缓冲区溢出，对于list、set、hash、zset类型的key，并不一定只使用一条命令；而是以某个常量为界将命令拆分为多条。

AOF重写的好处：

- 降低了文件占用空间
- 更小的AOF文件可以更快地被Redis加载 。

AOF重写分为手动触发和自动触发： 

- **手动触发**： 直接调用bgrewriteaof命令，该命令的执行与bgsave有些类似，都是fork子进程进行具体的工作，且都只有在fork时阻塞

- **自动触发**： 根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage参数，以及aof_current_size和aof_base_size状态确定触发时机 

**auto-aof-rewrite-min-size**：执行AOF重写时，文件的最小体积，默认值为64MB

**auto-aof-rewrite-percentage**：执行AOF重写时，当前AOF大小(即aof_current_size)和上一次重写时AOF大小(aof_base_size)的比值。

**自动触发时机**：aof_current_size>auto-aof-rewrite-minsize && (aof_current_size-aof_base_size) /aof_base_size>=auto-aof-rewrite-percentage 

**文件重写流程：**

![文件重写流程](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/%E6%96%87%E4%BB%B6%E9%87%8D%E5%86%99%E6%B5%81%E7%A8%8B.png)

上图是文件重写的流程：

1) Redis父进程首先判断当前是否存在正在执行 bgsave/bgrewriteaof的子进程，如果存在则bgrewriteaof命令直接返回，如果存在bgsave命令则等bgsave执行完成后再执行。前面介绍过，这个主要是基于性能方面的考虑。

2) 父进程执行fork操作创建子进程，这个过程中父进程是阻塞的。

3.1) 父进程fork后，bgrewriteaof命令返回”Background append only file rewrite started”信息并不再阻塞父进程，并可以响应其他命令。**Redis的所有写命令依然写入AOF缓冲区，并根据appendfsync策略同步到硬盘，保证原有AOF机制的正确。**

3.2) 由于fork操作使用写时复制技术，子进程只能共享fork操作时的内存数据。**由于父进程依然在响应命令，因此Redis使用AOF重写缓冲区(图中的aof_rewrite_buf)保存这部分数据，防止新AOF文件生成期间丢失这部分数据。也就是bgrewriteaof执行期间，Redis的写命令同时追加到aof_buf和aof_rewirte_buf两个缓冲区。**

4) 子进程根据内存快照，按照命令合并规则写入到新的AOF文件。

5.1) 子进程写完新的AOF文件后，向父进程发信号，父进程更新统计信息。

5.2) 父进程把AOF重写缓冲区的数据写入到新的AOF文件，这样就保证了新AOF文件所保存的数据库状态和服务器当前状态一致。

5.3) 使用新的AOF文件替换老文件，完成AOF重写。

### 4.6文件载入

![Redis持久化文件加载流程](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/redis/Redis%E6%8C%81%E4%B9%85%E5%8C%96%E6%96%87%E4%BB%B6%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B.png)

上图是redis文件载入流程：

1） AOF持久化开启且存在AOF文件时， 优先加载AOF文件

2） AOF关闭或者AOF文件不存在时， 加载RDB文件

3） 加载AOF/RDB文件成功后， Redis启动成功

4） AOF/RDB文件存在错误时， Redis启动失败并打印错误信息



### 4.7AOF常用配置

- **appendonly no**：是否开启AOF
- **appendfilename "appendonly.aof"**：AOF文件名
- **dir ./**：RDB文件和AOF文件所在目录
- **appendfsync everysec**：fsync持久化策略
- **no-appendfsync-on-rewrite no**：AOF重写期间是否禁止fsync；如果开启该选项，可以减轻文件重写时CPU和硬盘的负载，但是可能会丢失AOF重写期间的数据；需要在负载和安全性之间进行平衡
- **auto-aof-rewrite-percentage 100**：文件重写触发条件之一
- **auto-aof-rewrite-min-size 64MB**：文件重写触发提交之一
- **aof-load-truncated yes**：如果AOF文件结尾损坏，Redis启动时是否仍载入AOF文件

### 4.8AOF优缺点

**优点：**

- 使用 AOF 持久化会让 Redis 变得非常耐久，你可以设置不同的 `fsync` 策略，比如无 `fsync` ，每秒钟一次 `fsync` ，或者每次执行写入命令时 `fsync` 。 AOF 的默认策略为每秒钟 `fsync` 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（ `fsync` 会在后台线程执行，所以主线程可以继续努力地处理命令请求）。
- AOF 文件是一个只进行追加操作的日志文件， 即使日志因为某些原因而包含了未写入完整的命令（比如写入时磁盘已满，写入中途停机等等）， `redis-check-aof` 工具可以轻易地修复。
- Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写，重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。 整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。
- AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析也很轻松。 导出AOF 文件也非常简单， 举个例子， 如果你不小心执行了 [FLUSHALL](http://redisdoc.com/server/flushall.html#flushall) 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 [FLUSHALL](http://redisdoc.com/server/flushall.html#flushall) 命令， 并重启 Redis ， 就可以将数据集恢复到 [FLUSHALL](http://redisdoc.com/server/flushall.html#flushall) 执行之前的状态。

**缺点：**

- 对于相同的数据集来说，AOF 文件的体积一般要大于 RDB 文件的体积。
- 根据所使用的 `fsync` 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 `fsync` 的性能依然非常高， 而关闭 `fsync` 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间。



## 5.RDB与AOF 持久化如何选择

首先大家得有一个认识，无论是RDB还是AOF，持久化的开启都是要付出性能方面的代价：

- 对于RDB持久化，一方面是bgsave在进行fork操作时Redis主进程会阻塞，另一方面，子进程向硬盘写数据也会带来IO压力；

- 对于AOF持久化，向硬盘写数据的频率大大提高(everysec策略下为秒级)，IO压力更大，甚至可能造成AOF追加阻塞问题，此外，AOF文件的重写与RDB的bgsave类似，会有fork时的阻塞和子进程的IO压力问题。相对来说，由于AOF向硬盘中写数据的频率更高，因此对Redis主进程性能的影响会更大。

在实际生产环境中，根据数据量、应用对数据的安全要求、预算限制等不同情况，会有各种各样的持久化策略；如完全不使用任何持久化、使用RDB或AOF的一种，或同时开启RDB和AOF持久化等。此外，持久化的选择必须与Redis的主从策略一起考虑，因为主从复制与持久化同样具有数据备份的功能，而且主机master和从机slave可以独立的选择持久化方案。

 

下面分场景来讨论持久化策略的选择，下面的讨论也只是作为参考，实际方案可能更复杂更具多样性。

1) 如果Redis中的数据完全丢弃也没有关系（如Redis完全用作DB层数据的cache），那么无论是单机，还是主从架构，都可以不进行任何持久化。

2) 在单机环境下，如果可以接受十几分钟或更多的数据丢失，选择RDB对Redis的性能更加有利；如果只能接受秒级别的数据丢失，应该选择AOF。

3) 在多数情况下，我们都会配置主从环境，slave的存在既可以实现数据的热备，也可以进行读写分离分担Redis读请求，以及在master宕掉后继续提供服务。

在这种情况下，一种可行的做法是：

master：完全关闭持久化（包括RDB和AOF），这样可以让master的性能达到最好

slave：关闭RDB，开启AOF（如果对数据安全要求不高，开启RDB关闭AOF也可以），并定时对持久化文件进行备份（如备份到其他文件夹，并标记好备份的时间）；然后关闭AOF的自动重写，然后添加定时任务，在每天Redis闲时（如凌晨12点）调用bgrewriteaof。

为什么开启了主从复制，可以实现数据的热备份，还需要设置持久化呢？因为在一些特殊情况下，主从复制仍然不足以保证数据的安全，例如：

- master和slave进程同时停止：考虑这样一种场景，如果master和slave在同一栋大楼或同一个机房，则一次停电事故就可能导致master和slave机器同时关机，Redis进程停止；如果没有持久化，则面临的是数据的完全丢失。
- master误重启：考虑这样一种场景，master服务因为故障宕掉了，如果系统中有自动拉起机制（即检测到服务停止后重启该服务）将master自动重启，由于没有持久化文件，那么master重启后数据是空的，slave同步数据也变成了空的；如果master和slave都没有持久化，同样会面临数据的完全丢失。需要注意的是，即便是使用了**哨兵**进行自动的主从切换，也有可能在哨兵轮询到master之前，便被自动拉起机制重启了。因此，应尽量避免“自动拉起机制”和“不做持久化”同时出现。

4) 异地灾备：上述讨论的几种持久化策略，针对的都是一般的系统故障，如进程异常退出、宕机、断电等，这些故障不会损坏硬盘。但是对于一些可能导致硬盘损坏的灾难情况，如火灾地震，就需要进行异地灾备。例如对于单机的情形，可以定时将RDB文件或重写后的AOF文件，通过scp拷贝到远程机器，如阿里云、AWS等；对于主从的情形，可以定时在master上执行bgsave，然后将RDB文件拷贝到远程机器，或者在slave上执行bgrewriteaof重写AOF文件后，将AOF文件拷贝到远程机器上。一般来说，由于RDB文件文件小、恢复快，因此灾难恢复常用RDB文件；异地备份的频率根据数据安全性的需要及其他条件来确定，但最好不要低于一天一次。



## 6.参考文献

https://www.cnblogs.com/kismetv/p/9137897.html

http://redisdoc.com/topic/persistence.html

http://git.ym/panweijun/Redis/src/branch/master/Redis%E5%BC%80%E5%8F%91%E4%B8%8E%E8%BF%90%E7%BB%B4--%E4%BB%98%E7%A3%8A.pdf

http://git.ym/panweijun/Redis/src/branch/master/Redis%20%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf

