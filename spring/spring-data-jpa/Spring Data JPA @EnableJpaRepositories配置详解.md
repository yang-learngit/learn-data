# Spring Data JPA @EnableJpaRepositories配置详解

@EnableJpaRepositories注解用于Srping JPA的代码配置，用于取代xml形式的配置文件，@EnableJpaRepositories支持的配置形式丰富多用，本篇文章详细讲解。

## 简单配置

```java
@EnableJpaRepositories("com.spr.repository")
```

简单配置支持多个package，格式如下：

```java
@EnableJpaRepositories({"com.cshtong.sample.repository", "com.cshtong.tower.repository"})
```

 

## 单值和多组值配置方式

大部分注解可以都支持单个注解方式和多个注解，多个注解通常采用"{}"符号包含的一组数据。

比如：字符串形式的  "x.y.z"  =>  {"x.y.z","a.b.c"}

类别： A.class => {A.class, B.class}

 

## 完整的@EnableJpaRepositories注解

```java
@EnableJpaRepositories(
        // basePackages 支持多包扫描，用文本数组的形式就可以
        // 比如这样 {"com.simply.zuozuo.repo","com.simply.zuozuo.mapper"}
        basePackages = {
                "com.simply.zuozuo.repo"
        },
        value = {},
        // 指定里面的存储库类
        basePackageClasses = {},
        // 包含的过滤器
        includeFilters = {
                @ComponentScan.Filter(type = FilterType.ANNOTATION, value = Repository.class)
        },
        // 不包含的过滤器
        excludeFilters = {
                @ComponentScan.Filter(type = FilterType.ANNOTATION, value = Service.class),
                @ComponentScan.Filter(type = FilterType.ANNOTATION, value = Controller.class)
        },
        // 通过什么后缀来命名实现类，比如接口A的实现，名字叫AImpl
        repositoryImplementationPostfix = "Impl",
        // named SQL存放的位置，默认为META-INF/jpa-named-queries.properties
        namedQueriesLocation = "",
        // 枚举中有三个值，
        // CREATE_IF_NOT_FOUND，先搜索用户声明的，不存在则自动构建
        // USE_DECLARED_QUERY，用户声明查询
        // CREATE，按照接口名称自动构建查询
        queryLookupStrategy = QueryLookupStrategy.Key.CREATE_IF_NOT_FOUND,
        // 指定Repository的工厂类
        repositoryFactoryBeanClass = JpaRepositoryFactoryBean.class,
        // 指定Repository的Base类
        repositoryBaseClass = DefaultRepositoryBaseClass.class,
        // 实体管理工厂引用名称，对应到@Bean注解对应的方法
        entityManagerFactoryRef = "entityManagerFactory",
        // 事务管理工厂引用名称，对应到@Bean注解对应的方法
        transactionManagerRef = "transactionManager",
        // 是否考虑嵌套存储库
        considerNestedRepositories = false,
        // 开启默认事务
        enableDefaultTransactions = true
)
```



## 下面分别解释各个配置项的作用

### basePackage

用于配置扫描Repositories所在的package及子package。简单配置中的配置则等同于此项配置值，basePackages可以配置为单个字符串，也可以配置为字符串数组形式。

```java
@EnableJpaRepositories(
        basePackages = "com.cshtong")
```

多个包路径

```java
@EnableJpaRepositories(
        basePackages = {"com.cshtong.sample.repository", "com.cshtong.tower.repository"})
```

 

### basePackageClasses

指定 Repository 类

```java
@EnableJpaRepositories(basePackageClasses = BookRepository.class)
@EnableJpaRepositories(
        basePackageClasses = {ShopRepository.class, OrganizationRepository.class})
```

备注：测试的时候发现，配置包类的一个Repositories类，该包内其他Repositores也会被加载



### includeFilters

过滤器，该过滤区采用ComponentScan的过滤器类

```java
@EnableJpaRepositories(
        includeFilters={@ComponentScan.Filter(type=FilterType.ANNOTATION, value=Repository.class)})
```

 

### excludeFilters

不包含过滤器

```java
@EnableJpaRepositories(
        excludeFilters={
                @ComponentScan.Filter(type=FilterType.ANNOTATION, value=Service.class),
                @ComponentScan.Filter(type=FilterType.ANNOTATION, value=Controller.class)})
```

 

### repositoryImplementationPostfix

实现类追加的尾部，比如ShopRepository，对应的为ShopRepositoryImpl



### namedQueriesLocation

named SQL存放的位置，默认为META-INF/jpa-named-queries.properties



### queryLookupStrategy

构建条件查询的策略，包含三种方式CREATE，USE_DECLARED_QUERY，CREATE_IF_NOT_FOUND

- CREATE：按照接口名称自动构建查询
- USE_DECLARED_QUERY：用户声明查询
- CREATE_IF_NOT_FOUND：先搜索用户声明的，不存在则自动构建

该策略针对如下通过接口名称自动生成查询的场景

```java
  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String lastname);

  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String firstname);

  // Enabling static ORDER BY for a query
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname)
```



### repositoryFactoryBeanClass

指定Repository的工厂类



### entityManagerFactoryRef

实体管理工厂引用名称，对应到@Bean注解对应的方法

```java
     @Bean
     public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
         LocalContainerEntityManagerFactoryBean entityManagerFactoryBean = new LocalContainerEntityManagerFactoryBean();
         entityManagerFactoryBean.setDataSource(dataSource());
         entityManagerFactoryBean.setPersistenceProviderClass(HibernatePersistenceProvider.class);
         entityManagerFactoryBean
                 .setPackagesToScan(env.getRequiredProperty(PROPERTY_NAME_ENTITYMANAGER_PACKAGES_TO_SCAN));
         entityManagerFactoryBean.setJpaProperties(hibProperties());
         return entityManagerFactoryBean;
     }
```

 

### transactionManagerRef

事务管理工厂引用名称，对应到@Bean注解对应的方法

```java
@Bean 
public JpaTransactionManager transactionManager() {
    JpaTransactionManager transactionManager = new JpaTransactionManager();
    transactionManager.setEntityManagerFactory(entityManagerFactory().getObject());
    return transactionManager;
}
```