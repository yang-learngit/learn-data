# Canal数据库同步组件

本文主要描述Alibaba Canal中间件，官方文档请参考：

​    1）gitlab：https://github.com/alibaba/canal

​    2）主要原理介绍：https://github.com/alibaba/canal/wiki/canal%E4%BB%8B%E7%BB%8D

​    2）运维操作文档：https://github.com/alibaba/canal/wiki/AdminGuide

 

​    下文的介绍，基于大家对上述文档的基本了解！

​    1）Canal版本为：1.0.24

​    2）通过Canal同步数据库数据变更事件，并由下游的消费者消费，将数据转存到ES或者跨机房的DB中。

 

##     **一、设计目标**

​    1、监控canal组件以及客户端消费者

​    2、通过平台，能够实时查看监控数据。canal问题的定位应该快速，且运行状态数据可见。

​    3、按需提供报警策略。

​    4、平台支持添加canal集群的监控。

​    5、canal组件的部署和使用遵守约定，canal的实施应该快速。

  

​    我们希望构建一个canal服务：根据用户需求，能够快速构建canal集群，包括环境隔离；此外canal组件、上游的MySQL、下游的consumer等数据链路的整体状态都在监控之中，且数据可见。我们希望任何利益相关者，都可以参与到数据决策中，并按需提供报警、预警机制。

 

##     **二、基于Canal架构设计**

###     **1、整体架构**

   1）、每个Canal 集群应该至少有2个Canal实例，软硬件配置应该对等。我们不应该在同一个Cluster的多个节点上，配置有任何差异。

​    2）、一个Canal可以多个“instances”，每个instance对应一个“MySQL实例”的一个database（专业来说，一个instance对应一个MySQL实例，支持其上的多个databases）；简单而言，我们认为一个instance相当于一个逻辑Slave。

​    3）、由2、可以得出，每个Canal Instance的全局处理的数据总量与一个正常的MySQL Slave相同，如果保持同等SLA，从Canal instance角度考虑，它的硬件能力应该与MySQL Slave保持相同。（同为单线程处理）。

​    4）、原则上，每个Canal可以支持“数十个instance”，但是instance的个数最终会影响instance同步数据的效能。我们建议，一个Canal尽量保持一个instance；除非Slave数据变更极小，我们才会考虑合并instances，以提高Canal组件的利用效率。

​    5）、每个instance，一个单独的处理线程，用于负责“binlog dump”、“解析”、“入队和存储”。

​    6）、Canal集群模式，必须依赖Zookeeper，但是对Zookeeper的数据交互并不频繁。

​    7）、Canal集群运行态，为“M-S”模式。但是“M-S”的粒度为“instance”级别。如果当前Canal的instance，与MySQL建立连接并进行binlog解析时，发生一定次数的“网络异常”等，将会判定为当前instance失效，并stop（备注：此时会删除注册在ZK的相关临时节点）。同时，集群中的每个Canal都会注册所有“destination”（每个destination将有一个instance服务）的状态变更事件，如果“临时节点”被删除（或者不存在），则会出发抢占，抢占成功，则成为此instance的Master。

​    （源码：*CanalController.initGlobalConfig(),*

​        *ServerRunningMonitor.start(),*

​        *HeartBeatHAController.onFailed()*

​        ）

​    8）、根据7、，我们得知，如果Canal组件中有多个instances，有可能这些instances的Master会分布在不同的Canal节点上。

​    9）、在运维层面，我们基于“default-instance.xml”配置，基于“spring”模式；每个instance的配置，放置在各自的文件夹下。（${canal.root}/conf/${destination}/instance.properties）

​    10）、每个Canal节点，在启动时会初始化一个“嵌入式server”（NettyServer），此server主要目的是向Consumer提供服务。server的“ip:port”信息会注册在ZK中，此后Consumer通过ZK来感知。

​    （源码：

​         *ServerRunningMonitor.initRunning(),*

​         *ClusterNodeAccessStrategy构造方法，*

​         *ZookeeperPathUtils.getDestinationServerRunning(destination)*

​       ）

​    11）、在Canal运行期间，可以动态的增加instances配置、修改instances配置。

 

###     **2、Canal内部组件解析**





​    1）Canal节点，可以有多个instances，每个instance在运行时为一个单独的Spring Context，对象实例为“CanalInstanceWithSpring”。

​    2）每个instances有一个单独的线程处理整个数据流过程。

​    3）instance内部有EventParser、EventSink、EventStore、metaManager主要四个组件构成，当然还有其他的守护组件比如monitor、HA心跳检测、ZK事件监听等。对象实例初始化和依赖关系，可以参见“default-instance.xml”，其配置模式为普通的Spring。

​    （源码参见：*SpringCanalInstanceGenerator*）

​    4）Parser主要用于解析指定"数据库"的binlog，内部基于JAVA实现的“binlog dump”、“show master status”等。Parser会与ZK交互，并获取当前instance所有消费者的cursor，并获其最小值，作为此instance解析binlog的起始position。目前的实现，一个instance同时只能有一个consumer处于active消费状态，ClientId为定值“1001”，“cursor”中包含consumer消费binlog的position，数字类型。由此可见，Canal instance本身并没有保存binlog的position，Parser中继操作是根据consumer的消费cursor位置来决定；对于信息缺失时，比如Canal集群初次online，且在“default-instance.xml”中也没有指定“masterPositiion”信息(每个instance.properties是可以指定起始position的)，那么将根据“show master status”指令获取当前binlog的最后位置。

​    （源码：*MysqlEventParser.findStartPosition()*）

​    5）Parser每次、批量获取一定条数的binlog，将binlog数据封装成event，并经由EventSink将消息转发给EventStore，Sink的作用就是“协调Parser和Store”，确保binglog的解析速率与Store队列容量相容。

 

​    （参见源码：*AbstractEventParser.start(),*

​        *EntryEventSink.sink()*

​        ）

​    6）EventStore，用于暂存“尚未消费”的events的存储队列，默认基于内存的阻塞队列实现。Store中的数据由Sink组件提交入队，有NettyServer服务的消费者消费确认后出队，队列的容量和容量模式由“canal.properties”中的“memory”相关配置决定。当Store中容量溢满时，将会阻塞Sink操作（间接阻塞Parser），所以消费者的效能会直接影响instance的同步效率。

​    7）metaManager：主要用于保存Parser组件、CanalServer（即本文中提到的NettyServer）、Canal Instances的meta数据，其中Parser组件涉及到的binlog position、CanalServer与消费者交互时ACK的Cursor信息、instance的集群运行时信息等。根据官方解释，我们在production级别、高可靠业务要求场景下，metaManager建议基于Zookeeper实现。

​        其中有关Position信息由CanalLogPositionManager类负责，其实现类有多个，在Cluster模式下，建议基于FailbackLogPositionManager，其内部有“primary”、“failback”两级组合，优先基于primary来存取Position，只有当primary异常时会“降级”使用failback；其配置模式，建议与“default-instance.xml”保持一致。

​    （参看源码：*CanalMetaManager，PeriodMixedMetaManager*）

 

###     **3、Consumer端**

​    1）Consumer允许分布式部署，多个对等节点互备。但是任何时候，同一个destination的消费者只能有一个（client实例），这种排他、协调操作由zookeeper承担。在Cluster模式下，指定zkServer的地址，那么Consumer将会从meta信息中获取指定destination所对应的instance运行在哪个Canal节点上，且CanalServer（即NettyServer）的ip:port信息，那么此时Consumer将根据“ip:port”与NettyServer建立连接，并进行数据交互。

​    （参见源码：*SimpleCanalConnector.connect(),*

​        *ClientRunningMonitor.start()*

​        ）

​    2）Consumer有序消费消息，严格意义上说，我们强烈建议Consumer为单线程逐条处理。尽管研发同学，有很多策略可以让消息的处理过程使用多线程，但是对于消息的ACK将需要特殊的关注，而且非有序情境下，或许会对你的数据一致性有一定的影响。

​    3）消费者的消费效率，取决于“业务本身”，我们建议业务处理尽可能“短平快”。如果你的业务处理相对耗时，也不建议大家再使用“比如MQ、kafka”等其他异步存储做桥接，因为这本质上对提高endpoint端效能没有太大帮助，反而增加了架构的复杂性。

​    4）我们严格限制：消费者在处理业务时，必须捕获所有异常，并将异常的event和处理过程的exception打印到业务日志，以备将来进行数据补偿；捕获异常，有助于Consumer可以继续处理后续的event，那么整个canal链路不会因为一条消息而导致全部阻塞或者rollback。

​    5）Consumer单线程运行，阻塞、流式处理消息，获取event的方式为pull + batch；每个batch的size由配置决定，一个batch获取结束后，将会逐个调用业务的process方法，并在整个batch处理结束后，按需进行ack或者rollback。

​    6）需要注意：rollback操作是根据batchId进行，即回滚操作将会导致一个batch的消息会被重发；后续有重复消费的可能，这意味着业务需要有兼容数据幂等的能力。

​    7）消费者的ClientId为定值：1001，不可修改。

 

##     **三、部署与最佳实践（建议）**

###     **1、Canal集群部署**

​        1）Production场景，节点个数至少为2，考虑到Canal自身健壮性，也不建议Canal单组集群的节点数量过多。

​        2）Canal节点为“网络IO高耗”、“CPU高耗”（并发要求较高，体现在instance处理、consumer交互频繁）型应用，对磁盘IO、内存消耗很低。

​        3）不建议Canal与其他应用混合部署，我们认定Canal为核心组件，其可用性应该被保障在99.99%+。

​        4）每个Canal集群的instances个数，并没有严格限制，但其所能承载的数据量（TPS，包括consumer + binlog parser）是评估instances个数的主要条件。考虑到Production级别数据变更的场景不可控，我们建议每个Canal集群的instance个数，应该在1~3个。

​        5）对于核心数据库、TPS操作较高的数据库，应该使用单独的Canal。

​        6）Canal集群的个数多，或者分散，或者利用率低，并不是我们特别关注的事情，不要因为过度考虑“资源利用率”、“Consumer的集中化”而让Canal负重。

​        7）Canal的配置，绝大部分可以使用“默认”，但是要求在Production场景，instance模式必须使用Spring，配置方式采用“default-instance.xml”。“default-instance.xml”默认配置已满足我们HA环境下的所有设计要求。（版本：1.0.24）

​        8）Canal机器的配置要求（最低）：4Core、8G；建议：8Core、16G。

​        9）Canal的上游，即MySQL实例，可以是“Master”或者任意level的Slave，但是无论如何，其binlog_format必须为ROW，通过使用“show variables like 'binlog_format"”来确定。目前已经验证，使用mixed模式可能导致某些UPDATE操作事件无法被消费者解析的问题。

 

###     **2、Zookeeper集群**

​        1）Zookeeper集群，要求至少3个节点。网络联通性应该尽可能的良好。

​        2）多个Canal Cluster可以共享一个ZK集群，而且建议共享。那么只需要在canal.properties文件中“zkServers”配置项增加“rootPath”后缀即可，比如“10.0.1.21:2181,10.0.1.22:2181/canal/g1”。但是不同的Canal cluster，其rootPath应该不同。我们约定所有的Canal集群，rootpath的都以“/canal/”开头。（这对我们后续的ZK监控比较有利，我们只需要遍历"/canal"的子节点即可知道集群信息）

​        3）业界也有一种通用的部署方式，zookeeper集群与canal共生部署，三个节点，每个节点上都部署一个ZK和canal；这种部署模式的出发点也是比较简单，分析canal问题时只需要通过本地zk即可。（仅为建议）

​        4）需要非常注意，rootpath必须首先创建，否则canal启动时将会抛出异常！

 

###     **3、Consumer集群**

​        1）Consumer实例为普通application，JAVA项目，Spring环境。

​        2）Consumer集群至少2个节点，分布式部署。运行态为M-S。

​        3）每个Consumer实例为单线程，Consumer本身对CPU、内存消耗较低，但是对磁盘有一定的要求，因为我们将会打印大量的日志。建议磁盘为200G + ，logback的日志格式应该遵守我司规范，后续承接ELK基础数据平台。

​        4）一个Application中，允许有多个Consumer实例。

​        5）Consumer的业务处理部分，必须捕获全部异常，否则异常逃逸将可能导致整个链路的阻塞；对于异常情况下，建议进行日志记录，稍后按需进行数据补偿。

​        6）Consumer的业务处理部分，我们要求尽可能的快，业务处理简单；最重要的是千万不要在业务处理部分使用比如“Thread.sleep”、“Lock”等阻塞线程的操作，这可能导致主线程无法继续；如果必须，建议使用分支线程。

​        7）如果你对消息的顺序、事务不敏感，也允许你在业务处理部分使用多线程，这一部分有一定的歧义，所以需要开发者自己评估。从原理上说，多线程可以提高消息消费的效率，但是对数据一致性可能会有影响。但是Consumer的Client框架，仍然坚守单线程、有序交付。

​        8）在CanalServer和Consumer端，都能指定“filter”，即“过滤不关注的schema消息”；在CanalServer启动时将会首先加载“instance.properties”中的filter配置并生效，此后如果instance的消费者上线且也指定了filter，那么此filter信息将会被注册ZK中，那么CanalServer将会基于ZK获取此信息，并将Consumer端的filter作为最终决策；由此可见，我们在Consumer端指定filter的灵活性更高（当然隐蔽性也增加，这对排查问题需要一些提前沟通），无论如何，CanalServer不会传送“不符合filter”的消息给Consumer。

 

###     **4、Filter规则描述：**

​	适用于instance.properties和Consumer端的subscribe()方法

​        1）  所有表：.*   or  .*\\..*

​        2）  canal schema下所有表： canal\\..*

​        3）  canal下的以canal打头的表：canal\\.canal.*

​        4）  canal schema下的一张表：canal.test1

​        5）  多个规则组合使用：canal\\..*,mysql.test1,mysql.test2 (逗号分隔)

 

###     **5、运行状态监控**

​    非常遗憾的是，Canal监控能力相当的弱，内部程序中几乎没有JMX的任何export机制，所以如果需要监控比如“slave延迟”、“消费速率”、“position”等，需要开发代码。思路如下：

​    1）开发一个JAVA WEB项目。

​    2）读取ZK中的相关META信息，解析出每个destination对于的slave地址，并创建JDBC连接，发送“show master status”等指令，查看此slave binlog的位置，用于判断Canal延迟。

​    3）读取ZK中相关META信息，解析出每个destination对应的consumer cursor，与2）进行对比，用于判定consumer的消费延迟。

 

##     **四、Canal核心配置样例**

###     **1、canal.properties**

 (${canal.root}/conf)

```java
## 当前canal节点部署的instances列表，以“,”分割  
##比如：test,example  
canal.destinations= example  
##canal配置文件主目录，保持默认即可。  
##除非你为了提高canal的动态管理能力，将conf文件迁移到了其他目录（比如NFS目录等）  
canal.conf.dir = ../conf  
# 是否开启“instance”配置修改自动扫描和重载  
##1）conf.dir目录下新增、删除instance配置目录  
##2）instance配置目录下的instance.properties变更  
##不包含：canal.properties，spring/*.xml的配置变更  
##如果环境隔离、测试充分的环境下，或者应用试用初期，可以开启  
##对于高风险项目，建议关闭。  
canal.auto.scan = true  
canal.auto.scan.interval = 5  
   
   
##instance管理模式，Production级别我们要求使用spring  
canal.instance.global.mode = spring  
##直接初始化和启动instance  
canal.instance.global.lazy = false  
##Production级别，HA模式下，基于default-instance.xml  
##需要即备的ZK集群，且不应该修改此文件的默认配置。  
##如果有自定义的场景，应该新建${instance}-instance.xml文件  
canal.instance.global.spring.xml = classpath:spring/default-instance.xml  
   
   
   
##canal server的唯一标识，没有实际意义，但是我们建议同一个cluster上的不同节点，其ID尽可能唯一（后续升级）  
##数字类型  
canal.id = 1  
##canal server因为binding的本地IP地址，建议使用内网（唯一，集群可见，consumer可见）IP地址，比如“10.0.1.21”。  
#此IP主要为canalServer提供TCP服务而使用，将会被注册到ZK中,Consumer将与此IP建立连接。  
canal.ip =  
##conal server的TCP端口  
canal.port = 11111  
##Production场景，HA模式下，比如使用ZK作为服务管理，此处至少指定“多数派ZK Node”的IP列表  
##如果你的多个Canal Cluster共享ZK，那么每个Canal还需要使用唯一的“rootpath”。  
canal.zkServers = 10.0.1.21:2818,10.0.1.22,10.0.2.21:2818/canal/g1  
# flush data to zk  
##适用于metaManager，基于period模式  
##metaManager优先将数据（position）保存在内存，然后定时、间歇性的将数据同步到ZK中。  
##此参数用于控制同步的时间间隔，建议为“1000”（1S），单位：ms。  
##运维或者架构师，应该观察ZK的效能，如果TPS过于频繁，可以提高此值、或者按Canal集群分离ZK集群。  
##目前架构下，Consumer向CanalServer提交ACK时会导致ZK数据的同步。  
canal.zookeeper.flush.period = 1000  
##canal将parse、position数据写入的本地文件目录，HA环境下无效。  
##（file-instance.xml）  
canal.file.data.dir = ${canal.conf.dir}  
canal.file.flush.period = 1000  
##内存模式，EventStore为Memory类型时。（default-instance.xml）  
##可选值：  
##1) MEMSIZE   根据buffer.size * buffer.memunit的大小，限制缓存记录的大小，简答来说，就是内存容量大小限制  
##2) ITEMSIZE   根据buffer.size进行限制，简单来说，就是根据event的条数限制。  
##如果Canal上的instances个数有限，且Consumer的消费效率很高，甚至接近或者高于binlog解析效率，那么可以适度增加memory有关的数值。  
##此外batchMode还与消费者的batchSize有些关系，消费者每次能消费的数据量，取决于此mode。  
##如果mode为itemSize，则consumer每次获取的消息的条数为batchSize条。  
##如果mode为memSize，那么consumer消费的数据总量为batchSize * memunit  
canal.instance.memory.batch.mode = MEMSIZE  
canal.instance.memory.buffer.size = 16384  
canal.instance.memory.buffer.memunit = 1024  
   
   
# 所能支撑的事务的最大长度，超过阈值之后，一个事务的消息将会被拆分，并多次提交到eventStore中，但是将无法保证事务的完整性  
canal.instance.transaction.size =  1024  
# 当instance.properties配置文件中指定“master”、“standby”时，当canal与“master”联通性故障时，触发连接源的切换，  
##那么切换时，在新的mysql库上查找binlog时需要往前“回退”查找的时间，单位：秒。  
##良好架构下，我们建议不使用“standby”，限定一个数据库源。因为多个源时，数据库的调整频繁、协调不足，可能会引入一些数据问题。  
canal.instance.fallbackIntervalInSeconds = 60  
   
   
## 有关HA心跳检测部分，主要用在Parser管理dump连接时使用。  
## 我们在HA环境时建议开启。  
canal.instance.detecting.enable = true  
#如果你需要限定某个database的可用性验证（比如库锁），  
#最好使用复杂的、有效的SQL，比如：insert into {database}.{tmpTable} ....  
canal.instance.detecting.sql = select 1  
##心跳检测频率，单位秒  
canal.instance.detecting.interval.time = 6  
##重试次数  
##非常注意：interval.time * retry.threshold值，应该参考既往DBA同学对数据库的故障恢复时间，  
##“太短”会导致集群运行态角色“多跳”；“太长”失去了活性检测的意义，导致集群的敏感度降低，Consumer断路可能性增加。  
canal.instance.detecting.retry.threshold = 5  
#如果在instance.properties配置了“master”、“standby”，且此参数开启时，在“探测失败”后，会选择备库进行binlog获取  
#建议关闭  
canal.instance.detecting.heartbeatHaEnable = false  
   
   
# CanalServer、instance有关的TCP网络配置，建议保持抱人  
canal.instance.network.receiveBufferSize = 16384  
canal.instance.network.sendBufferSize = 16384  
canal.instance.network.soTimeout = 30  
   
# Parser组件，有关binlog解析的过滤  
##是否过滤dcl语句，比如“grant/create user”等  
canal.instance.filter.query.dcl = false  
##dml语句：insert/update/delete等  
canal.instance.filter.query.dml = false  
##ddl语句：create table/alter table/drop table以及一些index变更  
canal.instance.filter.query.ddl = false  
canal.instance.filter.table.error = false  
canal.instance.filter.rows = false  
   
   
# binlog格式和“镜像”格式检测，建议保持默认  
canal.instance.binlog.format = ROW,STATEMENT,MIXED  
canal.instance.binlog.image = FULL,MINIMAL,NOBLOB  
   
   
# ddl是否隔离发送，保持默认  
canal.instance.get.ddl.isolation = false 
```

   

​    canal.properties为全局配置，约束所有的instances、CanalServer等。

 

###     **2、instance.properties** 

（${canal.root}/conf/{instance}）

```java
## 每个instance都会伪装成一个mysql slave，  
## 考虑到binlog同步的机制，我们需要指定slaveId，注意此ID对于此canal前端的MySQL实例而言，必须是唯一的。  
##  同一个Canal cluster中相同instance，此slaveId应该一样。  
##  我们约定，所有Canal的instance，其slaveId以“1111”开头，后面补充四位数字。  
canal.instance.mysql.slaveId = 11110001  
   
# 数据库相关：master库  
##备注，master并不是要求是“MySQL 数据库Master”，  
## 而是Canal instance集群模式下，HA运行态中“master”（首选节点）  
## 当在故障恢复、Canal迁移时，我们需要手动指定binlog名称以及postition或者timestamp，确保新Canal不会丢失数据。  
## 数据库实例地址，ip:port  
canal.instance.master.address = 127.0.0.1:3306  
##指定起始的binlog文件名，保持默认  
canal.instance.master.journal.name =  
##此binlog文件的position位置（offset），数字类型。获取此position之后的数据。  
canal.instance.master.position =  
##此binlog的起始时间戳，获取此timestamp之后的数据。  
canal.instance.master.timestamp =  
   
##standby库  
##考虑到我司现状，暂不使用standby  
#canal.instance.standby.address =  
#canal.instance.standby.journal.name =  
#canal.instance.standby.position =  
#canal.instance.standby.timestamp =  
   
# 数据库连接的用户名和密码  
# 貌似Consumer与CanalServer建立连接时也用的是此用户名和密码  
canal.instance.dbUsername = canal  
canal.instance.dbPassword = canal  
# 默认数据库  
canal.instance.defaultDatabaseName =  
canal.instance.connectionCharset = UTF-8  
   
# schema过滤规则，类似于MySQL binlog的filter  
# canal将会过滤那些不符合要求的table，这些table的数据将不会被解析和传送  
# filter格式，Consumer端可以指定，只不过是后置的。  
## 无论是CanalServer还是Consumer，只要有一方指定了filter都会生效，consumer端如果指定，则会覆盖CanalServer端。  
canal.instance.filter.regex = .*\\..*  
# table black regex  
canal.instance.filter.black.regex =  
```



###     **3、default-instance.xml** 

（${canal.root}/conf/spring）

​    建议保持默认

 

##     **五、Consumer代码样例（抽象类）**

```java
package com.test.utils.canal;  
  
import com.alibaba.otter.canal.client.CanalConnector;  
import com.alibaba.otter.canal.client.CanalConnectors;  
import com.alibaba.otter.canal.protocol.CanalEntry;  
import com.alibaba.otter.canal.protocol.Message;  
import org.apache.commons.lang.SystemUtils;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
import org.slf4j.MDC;  
import org.springframework.beans.factory.InitializingBean;  
import org.springframework.util.CollectionUtils;  
  
import java.net.InetSocketAddress;  
import java.net.SocketAddress;  
import java.text.SimpleDateFormat;  
import java.util.Date;  
import java.util.List;  
  
/** 
 * Description 
 * <p> 
 * </p> 
 * DATE 17/10/19. 
 * 
 * @author liuguanqing. 
 */  
public abstract class AbstractCanalConsumer implements InitializingBean {  
  
  
    private static final Logger LOGGER = LoggerFactory.getLogger(AbstractCanalConsumer.class);  
    protected Thread.UncaughtExceptionHandler handler = new Thread.UncaughtExceptionHandler() {  
  
        public void uncaughtException(Thread t, Throwable e) {  
            LOGGER.error("parse events has an error", e);  
        }  
    };  
    protected static final String SEP = SystemUtils.LINE_SEPARATOR;  
    protected static String contextFormat = null;  
    protected static String rowFormat = null;  
    protected static String transactionFormat = null;  
    protected static final String DATE_FORMAT = "yyyy-MM-dd HH:mm:ss";  
  
    static {  
        StringBuilder sb = new StringBuilder();  
        sb.append(SEP)  
                .append("-------------Batch-------------")  
                .append(SEP)  
                .append("* Batch Id: [{}] ,count : [{}] , Mem size : [{}] , Time : {}")  
                .append(SEP)  
                .append("* Start : [{}] ")  
                .append(SEP)  
                .append("* End : [{}] ")  
                .append(SEP)  
                .append("-------------------------------")  
                .append(SEP);  
        contextFormat = sb.toString();  
  
        sb = new StringBuilder();  
        sb.append(SEP)  
                .append("+++++++++++++Row+++++++++++++>>>")  
                .append("binlog[{}:{}] , name[{},{}] , eventType : {} , executeTime : {} , delay : {}ms")  
                .append(SEP);  
        rowFormat = sb.toString();  
  
        sb = new StringBuilder();  
        sb.append(SEP)  
                .append("===========Transaction {} : {}=======>>>")  
                .append("binlog[{}:{}] , executeTime : {} , delay : {}ms")  
                .append(SEP);  
        transactionFormat = sb.toString();  
    }  
  
    private volatile boolean running = false;  
    protected Thread   thread;  
  
    private String zkServers;//cluster  
    private String address;//single，ip:port  
    private String destination;  
    private String username;  
    private String password;  
    private int batchSize = 1024;//  
  
    private String filter = "";//同canal filter，用于过滤database或者table的相关数据。  
  
    private boolean debug = false;//开启debug，会把每条消息的详情打印  
  
    /** 
     * 1:retry，重试，重试默认为3次，由retryTimes参数决定，如果重试次数达到阈值，则跳过，并且记录日志。 
     * 2:ignore,直接忽略，不重试，记录日志。 
     */  
    private int exceptionStrategy = 1;  
    private int retryTimes = 3;  
  
    private int waitingTime = 1000;//当binlog没有数据时，主线程等待的时间，单位ms,大于0  
  
    private CanalConnector connector;  
  
    public String getZkServers() {  
        return zkServers;  
    }  
  
    public void setZkServers(String zkServers) {  
        this.zkServers = zkServers;  
    }  
  
    public String getAddress() {  
        return address;  
    }  
  
    public void setAddress(String address) {  
        this.address = address;  
    }  
  
    public String getDestination() {  
        return destination;  
    }  
  
    public void setDestination(String destination) {  
        this.destination = destination;  
    }  
  
    public String getUsername() {  
        return username;  
    }  
  
    public void setUsername(String username) {  
        this.username = username;  
    }  
  
    public String getPassword() {  
        return password;  
    }  
  
    public void setPassword(String password) {  
        this.password = password;  
    }  
  
    public int getBatchSize() {  
        return batchSize;  
    }  
  
    public void setBatchSize(int batchSize) {  
        this.batchSize = batchSize;  
    }  
  
    public boolean isDebug() {  
        return debug;  
    }  
  
    public void setDebug(boolean debug) {  
        this.debug = debug;  
    }  
  
    public int getRetryTimes() {  
        return retryTimes;  
    }  
  
    public void setRetryTimes(int retryTimes) {  
        this.retryTimes = retryTimes;  
    }  
  
    public int getExceptionStrategy() {  
        return exceptionStrategy;  
    }  
  
    public void setExceptionStrategy(int exceptionStrategy) {  
        this.exceptionStrategy = exceptionStrategy;  
    }  
  
    public String getFilter() {  
        return filter;  
    }  
  
    public void setFilter(String filter) {  
        this.filter = filter;  
    }  
  
    public int getWaitingTime() {  
        return waitingTime;  
    }  
  
    public void setWaitingTime(int waitingTime) {  
        this.waitingTime = waitingTime;  
    }  
  
    /** 
     * 强烈建议捕获异常 
     * @param header 
     * @param afterColumns 
     */  
    public abstract void insert(CanalEntry.Header header,List<CanalEntry.Column> afterColumns);  
  
    /** 
     * 强烈建议捕获异常 
     * @param header 
     * @param beforeColumns 变化之前的列数据 
     * @param afterColumns 变化之后的列数据 
     */  
    public abstract void update(CanalEntry.Header header,List<CanalEntry.Column> beforeColumns,List<CanalEntry.Column> afterColumns);  
  
    /** 
     * 强烈建议捕获异常 
     * @param header 
     * @param beforeColumns 删除之前的列数据 
     */  
    public abstract void delete(CanalEntry.Header header,List<CanalEntry.Column> beforeColumns);  
  
    /** 
     * 创建表 
     * @param header 可以从header中获得schema、table的名称 
     * @param sql 
     */  
    public void createTable(CanalEntry.Header header,String sql) {  
        String schema = header.getSchemaName();  
        String table = header.getTableName();  
        LOGGER.info("parse event,create table,schema: {},table: {},SQL: {}",new String[] {schema,table,sql});  
    }  
  
    /** 
     * 修改表结构,即alter指令，需要声明：通过alter增加索引、删除索引，也是此操作。 
     * @param header 可以从header中获得schema、table的名称 
     * @param sql 
     */  
    public void alterTable(CanalEntry.Header header,String sql) {  
        String schema = header.getSchemaName();  
        String table = header.getTableName();  
        LOGGER.info("parse event,alter table,schema: {},table: {},SQL: {}",new String[] {schema,table,sql});  
    }  
  
    /** 
     * 清空、重建表 
     * @param header 可以从header中获得schema、table的名称 
     * @param sql 
     */  
    public void truncateTable(CanalEntry.Header header,String sql) {  
        String schema = header.getSchemaName();  
        String table = header.getTableName();  
        LOGGER.info("parse event,truncate table,schema: {},table: {},SQL: {}",new String[] {schema,table,sql});  
    }  
  
    /** 
     * 重命名schema或者table，注意 
     * @param header 可以从header中获得schema、table的名称 
     * @param sql 
     */  
    public void rename(CanalEntry.Header header,String sql) {  
        String schema = header.getSchemaName();  
        String table = header.getTableName();  
        LOGGER.info("parse event,rename table,schema: {},table: {},SQL: {}",new String[] {schema,table,sql});  
    }  
  
    /** 
     * 创建索引,通过“create index on table”指令 
     * @param header 可以从header中获得schema、table的名称 
     * @param sql 
     */  
    public void createIndex(CanalEntry.Header header,String sql) {  
        String schema = header.getSchemaName();  
        String table = header.getTableName();  
        LOGGER.info("parse event,create index,schema: {},table: {},SQL: {}",new String[] {schema,table,sql});  
    }  
  
    /** 
     * 删除索引，通过“delete index on table”指令 
     * @param header      * 可以从header中获得schema、table的名称 
     * @param sql 
     */  
    public void deleteIndex(CanalEntry.Header header,String sql) {  
        String schema = header.getSchemaName();  
        String table = header.getTableName();  
        LOGGER.info("parse event,delete table,schema: {},table: {},SQL: {}",new String[] {schema,table,sql});  
    }  
  
    /** 
     * 强烈建议捕获异常，非上述已列出的其他操作，非核心 
     * 除了“insert”、“update”、“delete”操作之外的，其他类型的操作. 
     * 默认实现为“无操作” 
     * @param entry 
     */  
    public void whenOthers(CanalEntry.Entry entry) {  
    }  
  
  
    @Override  
    public void afterPropertiesSet() throws Exception {  
        if(waitingTime <= 0 ) {  
            throw new IllegalArgumentException("waitingTime must be greater than 0");  
        }  
        if(ExceptionStrategy.codeOf(exceptionStrategy) == null) {  
            throw new IllegalArgumentException("exceptionStrategy is not valid,1 or 2");  
        }  
        start();  
    }  
  
    public synchronized void start() {  
        if(running) {  
            return;  
        }  
  
        if(zkServers != null && zkServers.length() > 0) {  
            connector = CanalConnectors.newClusterConnector(zkServers,destination,username,password);  
        } else if (address != null){  
            String[] segments = address.split(":");  
            SocketAddress socketAddress = new InetSocketAddress(segments[0],Integer.valueOf(segments[1]));  
            connector = CanalConnectors.newSingleConnector(socketAddress,destination,username,password);  
        } else {  
            throw new IllegalArgumentException("zkServers or address cant be null at same time,you should specify one of them!");  
        }  
  
        thread = new Thread(new Runnable() {  
  
            public void run() {  
                process();  
            }  
        });  
  
        thread.setUncaughtExceptionHandler(handler);  
        thread.start();  
        running = true;  
    }  
  
  
    protected synchronized void stop() {  
        if (!running) {  
            return;  
        }  
        running = false;//process()将会在下一次loop时退出  
        if (thread != null) {  
            thread.interrupt();  
            try {  
                thread.join();  
            } catch (InterruptedException e) {  
                // ignore  
            }  
        }  
        MDC.remove("destination");  
    }  
    /** 
     * 
     * 用于控制当连接异常时，重试的策略，我们不应该每次都是立即重试，否则将可能导致大量的错误，在空转时导致CPU过高的问题 
     * sleep策略基于简单的累加，最长不超过3S 
     */  
    private void sleepWhenFailed(int times) {  
        if(times <= 0) {  
            return;  
        }  
        try {  
            int sleepTime = 1000 + times * 100;//最大sleep 3s。  
            Thread.sleep(sleepTime);  
        } catch (Exception ex) {  
            //  
        }  
    }  
  
    protected void process() {  
        int times = 0;  
        while (running) {  
            try {  
                sleepWhenFailed(times);  
                //after block,should check the status of thread.  
                if(!running) {  
                    break;  
                }  
                MDC.put("destination", destination);  
                connector.connect();  
                connector.subscribe(filter);  
                times = 0;//reset;  
                while (running) {  
                    Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据，不确认  
                    long batchId = message.getId();  
                    int size = message.getEntries().size();  
                    if (batchId == -1 || size == 0) {  
                        try {  
                            Thread.sleep(waitingTime);  
                        } catch (InterruptedException e) {  
                            //  
                        }  
                        continue;  
                    }  
                    //logger  
                    printBatch(message, batchId);  
  
                    //遍历每条消息  
                    for(CanalEntry.Entry entry : message.getEntries()) {  
                        session(entry);//no exception  
                    }  
                    //ack all the time。  
                    connector.ack(batchId);  
                }  
            } catch (Exception e) {  
                LOGGER.error("process error!", e);  
                if(times > 20) {  
                    times = 0;  
                }  
                times++;  
            } finally {  
                connector.disconnect();  
                MDC.remove("destination");  
            }  
        }  
    }  
  
    protected void session(CanalEntry.Entry entry) {  
        CanalEntry.EntryType entryType = entry.getEntryType();  
        int times = 0;  
        boolean success = false;  
        while (!success) {  
            if(times > 0) {  
                /** 
                 * 1:retry，重试，重试默认为3次，由retryTimes参数决定，如果重试次数达到阈值，则跳过，并且记录日志。 
                 * 2:ignore,直接忽略，不重试，记录日志。 
                 */  
                if (exceptionStrategy == ExceptionStrategy.RETRY.code) {  
                    if(times >= retryTimes) {  
                        break;  
                    }  
                } else {  
                    break;  
                }  
            }  
            try {  
                switch (entryType) {  
                    case TRANSACTIONBEGIN:  
                        transactionBegin(entry);  
                        break;  
                    case TRANSACTIONEND:  
                        transactionEnd(entry);  
                        break;  
                    case ROWDATA:  
                        rowData(entry);  
                        break;  
                    default:  
                        break;  
                }  
                success = true;  
            } catch (Exception e) {  
                times++;  
                LOGGER.error("parse event has an error ,times: + " + times + ", data:" + entry.toString(), e);  
            }  
  
        }  
  
        if(debug && success) {  
            LOGGER.info("parse event success,position:" + entry.getHeader().getLogfileOffset());  
        }  
    }  
  
    private void rowData(CanalEntry.Entry entry) throws Exception {  
        CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());  
        CanalEntry.EventType eventType = rowChange.getEventType();  
        CanalEntry.Header header = entry.getHeader();  
        long executeTime = header.getExecuteTime();  
        long delayTime = new Date().getTime() - executeTime;  
        String sql = rowChange.getSql();  
        if(debug) {  
            if (eventType == CanalEntry.EventType.QUERY || rowChange.getIsDdl()) {  
                LOGGER.info("------SQL----->>> type : {} , sql : {} ", new Object[]{eventType.getNumber(), sql});  
            }  
            LOGGER.info(rowFormat,  
                    new Object[]{  
                            header.getLogfileName(),  
                            String.valueOf(header.getLogfileOffset()),  
                            header.getSchemaName(),  
                            header.getTableName(),  
                            eventType,  
                            String.valueOf(executeTime),  
                            String.valueOf(delayTime)  
                    });  
        }  
  
        try {  
            //对于DDL，直接执行，因为没有行变更数据  
            switch (eventType) {  
                case CREATE:  
                    createTable(header,sql);  
                    return;  
                case ALTER:  
                    alterTable(header,sql);  
                    return;  
                case TRUNCATE:  
                    truncateTable(header,sql);  
                    return;  
                case ERASE:  
                    LOGGER.debug("parse event : erase,ignored!");  
                    return;  
                case QUERY:  
                    LOGGER.debug("parse event : query,ignored!");  
                    return;  
                case RENAME:  
                    rename(header,sql);  
                    return;  
                case CINDEX:  
                    createIndex(header,sql);  
                    return;  
                case DINDEX:  
                    deleteIndex(header,sql);  
                    return;  
                default:  
                    break;  
            }  
            //对于有行变更操作的  
            for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {  
                switch (eventType) {  
                    case DELETE:  
                        delete(header, rowData.getBeforeColumnsList());  
                        break;  
                    case INSERT:  
                        insert(header, rowData.getAfterColumnsList());  
                        break;  
                    case UPDATE:  
                        update(header, rowData.getBeforeColumnsList(), rowData.getAfterColumnsList());  
                        break;  
                    default:  
                        whenOthers(entry);  
                }  
            }  
        } catch (Exception e) {  
            LOGGER.error("process event error ,",e);  
            LOGGER.error(rowFormat,  
                    new Object[]{  
                            header.getLogfileName(),  
                            String.valueOf(header.getLogfileOffset()),  
                            header.getSchemaName(),  
                            header.getTableName(),  
                            eventType,  
                            String.valueOf(executeTime),  
                            String.valueOf(delayTime)  
                    });  
            throw e;//重新抛出  
        }  
    }  
  
    /** 
     * default，only logging information 
     * @param entry 
     */  
    public void transactionBegin(CanalEntry.Entry entry) {  
        if(!debug) {  
            return;  
        }  
        try {  
            CanalEntry.TransactionBegin begin = CanalEntry.TransactionBegin.parseFrom(entry.getStoreValue());  
            // 打印事务头信息，执行的线程id，事务耗时  
            CanalEntry.Header header = entry.getHeader();  
            long executeTime = header.getExecuteTime();  
            long delayTime = new Date().getTime() - executeTime;  
            LOGGER.info(transactionFormat,  
                    new Object[] {  
                            "begin",  
                            begin.getTransactionId(),  
                            header.getLogfileName(),  
                            String.valueOf(header.getLogfileOffset()),  
                            String.valueOf(header.getExecuteTime()),  
                            String.valueOf(delayTime)  
                    });  
        } catch (Exception e) {  
            LOGGER.error("parse event has an error , data:" + entry.toString(), e);  
        }  
    }  
  
    public void transactionEnd(CanalEntry.Entry entry) {  
        if(!debug) {  
            return;  
        }  
        try {  
            CanalEntry.TransactionEnd end = CanalEntry.TransactionEnd.parseFrom(entry.getStoreValue());  
            // 打印事务提交信息，事务id  
            CanalEntry.Header header = entry.getHeader();  
            long executeTime = header.getExecuteTime();  
            long delayTime = new Date().getTime() - executeTime;  
            LOGGER.info(transactionFormat,  
                    new Object[]{  
                            "end",  
                            end.getTransactionId(),  
                            header.getLogfileName(),  
                            String.valueOf(header.getLogfileOffset()),  
                            String.valueOf(header.getExecuteTime()),  
                            String.valueOf(delayTime)  
                    });  
        } catch (Exception e) {  
            LOGGER.error("parse event has an error , data:" + entry.toString(), e);  
        }  
    }  
  
  
    /** 
     * 打印当前batch的摘要信息 
     * @param message 
     * @param batchId 
     */  
    protected void printBatch(Message message, long batchId) {  
        List<CanalEntry.Entry> entries = message.getEntries();  
        if(CollectionUtils.isEmpty(entries)) {  
            return;  
        }  
  
        long memSize = 0;  
        for (CanalEntry.Entry entry : entries) {  
            memSize += entry.getHeader().getEventLength();  
        }  
        int size = entries.size();  
        String startPosition = buildPosition(entries.get(0));  
        String endPosition = buildPosition(message.getEntries().get(size - 1));  
  
        SimpleDateFormat format = new SimpleDateFormat(DATE_FORMAT);  
        LOGGER.info(contextFormat, new Object[] {  
                batchId,  
                size,  
                memSize,  
                format.format(new Date()),  
                startPosition,  
                endPosition }  
        );  
    }  
  
    protected String buildPosition(CanalEntry.Entry entry) {  
        CanalEntry.Header header = entry.getHeader();  
        long time = header.getExecuteTime();  
        Date date = new Date(time);  
        SimpleDateFormat format = new SimpleDateFormat(DATE_FORMAT);  
        StringBuilder sb = new StringBuilder();  
        sb.append(header.getLogfileName())  
                .append(":")  
                .append(header.getLogfileOffset())  
                .append(":")  
                .append(header.getExecuteTime())  
                .append("(")  
                .append(format.format(date))  
                .append(")");  
        return sb.toString();  
    }  
  
    enum ExceptionStrategy {  
        RETRY(1),  
        IGNORE(2);  
        int code;  
        ExceptionStrategy(int code) {  
            this.code = code;  
        }  
        public static ExceptionStrategy codeOf(Integer code) {  
            if(code == null) {  
                return null;  
            }  
            for(ExceptionStrategy e : ExceptionStrategy.values()) {  
                if(e.code == code) {  
                    return e;  
                }  
            }  
            return null;  
        }  
    }  
}  
```



​    备注：如果基于springboot或者其他方式实例化CanalConsumer，需要显示的执行“start()”方法；且在Spring容器关闭时，建议执行“stop()”方法，让Canal平滑关闭

```java
<bean id="sampleCanalConsumer" class="com.test.demo.canal.SampleCanalConsumer" destroy-method="stop">  
    <property name="zkServers" value="10.0.1.21:2181,10.1.2.21:2181/canal/g1"/>  
    <property name="batchSize" value="2048" />  
    <property name="debug" value="true"/>  
    <property name="destination" value="sample"/>  
    <property name="username" value="canal"/>  
    <property name="password" value="canal"/>  
    <property name="exceptionStrategy" value="1"/>  
    <property name="retryTimes" value="3"/>  
    <!--  
    <property name="filter" value="sample.t1,sample.t2" />  
    -->  
</bean>  
```



##    **六、META数据整理**

（zookeeper，省略了chrootpath）

​    *1、/otter/canal/cluster*：子节点列表，临时节点

​    含义：表示当前集群中处于Active状态的，CanalServer的列表和TCP端口。

​    创建：当CanalServer节点启动时创建。

​    删除：CanalServer失效时删除对应的临时节点。

 

​    *2、/otter/canal/destinations/*：子节点列表

​    含义：表示当前集群中，正在提供服务的instances对应的destination列表。根据原理，每个destination对应一个instance实例。

​    创建：Canal节点初始化instance时。

​    删除：当instance配置被删除时。

 

​    *3、/otter/canal/destinations/${destination}/running*：节点值，临时节点

​    含义：表示此destination对应的instances由哪个CanalServer提供服务。

​    创建：当次instance初始化时，通过ZK抢占成功后，写入Server信息。集群中多个Canal节点会同时初始化instance，但是只有一个Canal处于服务状态。

​    获取：Consumer会根据destination从ZK中获取此信息，并根据”address“与CanalServer建立连接。

​    删除：instance无法与MySQL Server正常通讯、且重试N次后，将会触发HA切换，此时会删除此节点。同时，其他Canal节点会检测ZK事件，并重新抢占，成功者将会成为此destination的服务者。

 

​    *4、/otter/canal/destinations/${destination}/cluster*：子节点列表，临时节点

​    含义：集群中可以提供此destination服务的CanalServer列表（不表示正在提供服务的CannalServer）。

​    创建：instance初始化时

​    获取：Consumer侦听此列表的变更事件。

 

​    *5、/otter/canal/destinations/${destination}/1001*：节点存在与否，历史节点

​    含义：“1001”为定值，如果存在此子节点，表示此destination上有Active状态的消费者。

​    创建：Consumer启动时。

​    获取：Consumer集群的每个实例都会探测其变更事件。

 

​    *6、/otter/canal/destinations/${destination}/1001/filter*：节点值

​    含义：表示此destination使用的过滤器。

​    创建：1）instance初始化时，会读取instance.properties文件中的filter配置，并保存在此节点值中。 2）Consumer初始化时，由subscribe()方法传入，并通过RPC同学发送给CanalServer并由其修改此值。

​    获取：instance中的Parser组件会侦听此值的变更，并根据此值作为binlog过滤的条件。

 

​    *7、/otter/canal/destinations/${destination}/1001/cursor*：节点值

​    含义：其“position”字段表示Consumer已经消费、ACK的binglog位置。“timestamp”表示最后位置的binlog事件发生的时间。可以通过此节点值，判断当前Canal的延迟。

​    创建：CanalServer收到Consumer ACK之后，有定时器间歇性的写入ZK。（metaManager，参见default-instance.xml）

 

​    *8、/otter/canal/destinations/${destination}/1001/running*：节点值，临时节点

​    含义：当前处于running状态的Consumer实例所在的位置。

​    创建：Consumer实例初始化，且抢占成功时。

​    获取：其他Consumer会检测节点的变更事件。

​    删除：Consumer实例关闭时。

 

​    **七、Canal Instance位点查找过程（spring模式）**

​     1、从ZK中获取消费者ACK的位点LogPosition，包括binlog名称和position、时间戳等。

​     2、如果1、存在，则比较LogPosition与当前配置中的"master.address"地址是否一致：

​            1）如果一致，且dump正常，则使用此位点。但是如果dump错误次数达到阈值（dumpErrorCountThreshold），则使用LogPosition中的timestamp回退一分钟，重新获取位点（如果获取失败，则启动失败）。（根据timestamp获取Position的方式，就是将binlog文件列表中查找此时间戳所在的binlog文件，并重新构建LogPosition，如果找不到则返回null，后续会中断instance服务）

​            2）如果不一致，表示切换了数据库地址，则直接将timestamp回退一分钟，从新库（当前配置的“master.address”）获取新的LogPosition并使用。

​     3、如果1、不存在，表示没有历史消费记录，依次从“master.address” 、“standby.address”配置中选择一个不为空的作为LogPosition。

​            1）如果LogPosition中，没有指定journalName（即binlog文件名），则使用timestamp获取位点（方式同上）；如果也没有指定timestamp，则直接使用数据库最新位点。

​            2）如果指定了journalName，此时也明确指定了“position”配置参数，则直接使用；如果没有指定“position”但是指定了timestamp，则尝试获取实际的position并使用，如果在此timestamp没有找到位点，则从此journalName中首条开始。

 

​     无论是ZK获取position、还是通过instance.properties获取postition，信息都应该正确与数据库实际状态匹配；否则将无法构建LogPosition对象，直接导致此instance停止服务。

 

​    **八、答疑**

​    **1、Canal会不会丢失数据？**

​    答：Canal正常情况下不会丢失数据，比如集群节点失效、重启、Consumer关闭等；但是，存在丢数据的风险可能存在如下几种可能：

​        1）ZK的数据可靠性或者安全性被破坏，比如ZK数据丢失，ZK的数据被人为串改，特别是有关Position的值。

​        2）MySQL binlog非正常运维，比如binglog迁移、重命名、丢失等。

​        3）切换MySQL源，比如原来基于M1实例，后来M1因为某种原因失效，那么Canal将数据源切换为M2，而且M1和M2可能binlog数据存在不一致（非常有可能）。

​        4）Consumer端ACK的时机不佳，比如调用get()方法，而不是getWithoutAck()，那么消息有可能尚未完全消费，就已经ACK，那么此时由异常或者Consumer实例失效，则可能导致消息丢失。我们需要在ACK时机上保障“at lease once”。

 

​    **2、Canal的延迟很大是什么原因？**

​    答：根据数据流的pipeline，“Master” > "Slave" > "Canal" > "Consumer"，每个环节都需要耗时，而且整个管道中都是单线程、串行、阻塞式。（假如网络层面都是良好的）

​        1）如果批量insert、update、delete，都可能导致大量的binlog产生，也会加剧Master与slave之间数据同步的延迟。（写入频繁）

​        2）“Consumer”消费的效能较低，比如每条event执行耗时很长。这会导致数据变更的消息ACK较慢，那么对于Canal而言也将阻塞，直到Canal内部的store有足够的空间存储新消息、才会继续与Slave进行数据同步。

​        3）如果Canal节点ZK的网络联通性不畅，将会导致Canal集群处于动荡状态，大量的时间消耗在ZK状态监测和维护上，而无法对外提供正常服务，包括不能顺畅的dump数据库数据。

 

​    **3、Canal会导致消息重复吗？**

​    答：会，这从两个大的方面谈起。

​        1）Canal instance初始化时，根据“消费者的Cursor”来确定binlog的起始位置，但是Cursor在ZK中的保存是滞后的（间歇性刷新），所以Canal instance获得的起始position一定不会大于消费者真实已见的position。

​        2）Consumer端，因为某种原因的rollback，也可能导致一个batch内的所有消息重发，此时可能导致重复消费。

​    我们建议，Consumer端需要保持幂等，对于重复数据可以进行校验或者replace。对于非幂等操作，比如累加、计费，需要慎重。

 

​    **4、Canal性能如何？**

​    答：Canal本身非常轻量级，主要性能开支就是在binlog解析，其转发、存储、提供消费者服务等都很简单。它本身不负责数据存储。原则上，canal解析效率几乎没有负载，canal的本身的延迟，取决于其与slave之间的网络IO。

 

​    **5、Canal数据的集散问题，一个destination的消息能否被多个Consumer集群并行消费？**

​    答：比如有两个Consumer集群，C1/C2，你希望C1和C2中的消费者都能够订阅到相同的消息，就像Kafka或者JMS Topic一样...但是非常遗憾，似乎Canal无法做到，这取决于Canal内部的存储模式，Canal内部是一个“即发即失”的内存队列，无法权衡、追溯不同Consumer之间的消息，所以无法支持。

​        如果希望达到这种结果，有2个办法：第一，消费者收到消息以后转发到kafka或者MQ中，后继的其他Consumer只与kafka或者MQ接入；第二：一个Canal中使用多个destination，但是它们对应相同的MySQL源。

 

​    **6、我的Consumer从canal消费数据，但是我的业务有反查数据库的操作，那么数据一致性怎么做？**

​    答：从基本原理，我们得知canal就像一个“二级Slave”一样，所以canal接收到的数据总是相对滞后，如果消费者消费效率较低，那么从consumer的角度来说，它接收的数据更加滞后；如果consumer中反查数据库，无论它查找master还是其他任意level的从库，都会获得比当前视图更新（fresh）的数据，无论如何，我们总是无法做到完全意义上的“数据一致性”视图。

​     比如，canal消费者收到的数据为db.t1.row1.column1 = A，那么此时master上column1值已经更改为B，但是Slave可能因为与master同步延迟问题，此时Slave上column1值可能为C。所以无论你怎么操作，都无法得到一致性的数据。（数据发生的时间点，A < C < B）。

​     我们需要接受这种问题，为了避免更多干扰，consumer反查数据时使用canal所对应的slave可以在一定程度上缓解数据一致性的风险，但是这仍然无法解决问题。但是这种策略仍然有风险，会知道canal所对应的slave性能消耗加剧，进而增加数据同步的延迟。

​    理想的解决办法：canal的消费者，消费数据以后，写入到一个数据库或者ES，那么在消费者内部的数据反查操作，全部基于这个数据库或者ES。

 

​    **7、Consumer端无法进行消费的问题？**

​    答： 1）Consumer会与ZK集群保持联通性，用于检测消费者集群、CanalServer集群的变化，如果Consumer与ZK集群的联通性失效，将会导致消费者无法正常工作。

​    2）Consumer会与CanalServer保持TCP长连接，此长连接用于传输消息、心跳检测等，如果Consumer与CanalServer联通性故障，将有可能导致Consumer不断重试，此期间消息无法正常消费。

​    3）如果CanalServer与ZK联通性失效，将会导致此CanalServer释放资源，进行HA切换，切换时间取决于ZK的session活性检测，大概为30S，此期间消费者无法消费。

​    4）CanalServer中某个instance与slave联通性失效，将会触发HA切换，切换时间取决于HA心跳探测时间，大概为30S，此期间消费者无法消费。

​    

​    **8、如果Canal更换上游的master（或者slave），该怎么办？（比如迁库、迁表等）**

​    答：背景要求，我们建议“新的数据库最好是旧的数据库的slave”或者“新、旧数据库为同源master”，平滑迁移；

​        1）创建一个新的instance，使用新的destination，并与新的Slave创建连接。

​        2）在此期间，Consumer仍然与旧的destination消费。

​        3）通过“timestamp”确认，新的slave的最近binlog至少已经超过此值。

​        4）Consumer切换，使用新的destination消费，可能会消费到重复数据，但是不会导致数据丢失。

​    当然，更简单的办法就是直接将原destination中的数据库地址跟新即可，前提是新、旧两个数据库同源master，新库最好已经同步执行了一段时间。

 

​    **9、Canal如何重置消费的position？**

​    答：比如当消费者在消费binlog时，数据异常，需要回溯到旧的position重新消费，是这个场景！

​    1）我们首先确保，你需要回溯的position所对应的binlog文件仍然存在，可以通过需要回溯的时间点来确定position和binlog文件名，这一点可以通过DBA来确认。

​    2）关闭消费者，否则重置位点操作无法生效。（你可以在关闭消费者之前执行unsubscribe，来删除ZK中历史位点的信息）

​    3）关闭Canal集群，修改对应的destination下的配置文件中的“canal.instance.master.journal.name = <此position对应的binlog名称>”、“canal.instance.master.position = <此position>”；可以只需要修改一台。

​    4）删除zk中此destination的消费者meta信息，“${destination}/1001"此path下所有的子节点，以及“1001”节点。（可以通过消费者执行unsubscribe来实现）

​    5）重启2）中的此canal节点，观察日志。

​    6）重启消费者。

​    当然应该还有其他更优的操作方式，请大家补充。上述操作过程，是被验证可行的。

















