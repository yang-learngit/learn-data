Springboot使用logback写日志接入ELK

1、logstash的配置文件：

```conf
input {
    tcp {
        port => 4560
        codec => json_lines
    }
}
output{
    elasticsearch { 
        hosts => ["localhost:9200"] 
        index => "%{[appName]}-%{+YYYY.MM.dd}" #用一个项目名称来做索引
	}
	stdout { codec => rubydebug }
}
```

配置说明：

- 4560 是logstash接收数据的端口
- codec => json_lines是一个json解析器，接收json的数据。这个要装 `logstash-codec-json_lines` 插件
- ouput  elasticsearch指向我们安装的地址

2、新建springboot项目

代码地址：https://github.com/yang-zhijiang/elkTest.git

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.test</groupId>
    <artifactId>springboot-with-elk</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>springboot-with-elk</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>5.1</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

新建启动类

```java
@SpringBootApplication
public class ElkTestApplication implements CommandLineRunner {
    public static void main(String[] args) {
        SpringApplication.run(ElkTestApplication.class, args);
    }
    @Override
    public void run(String... args) throws Exception {
        Logger logger = LoggerFactory.getLogger(ElkTestApplication.class);
        logger.info("测试log");

        for (int i = 0; i < 10; i++) {
            logger.error("something wrong. id={}; name=Ryan-{};", i, i);
        }
    }
}
```

在resources下新建logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration>
<configuration>
  <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>localhost:4560</destination>
    <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder" />
  </appender>
  <include resource="org/springframework/boot/logging/logback/base.xml"/>
  <root level="INFO">
    <appender-ref ref="LOGSTASH" />
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```

