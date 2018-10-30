# IDEA通过JPA生成带注解的实体类详细步骤

## 连接数据库

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E5%BB%BA%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5.png)



打开项目：

1、点击右侧的datesource图标，要是没有该图标，请去自行百度

2、点击 + 号

3、选择 datasource

4、选择 mysql

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5%E5%86%85%E5%AE%B9.png)

1、填写一个连接名，随便填什么都行

2、不用选择，默认就行

3、填写数据库连接的 IP地址，比如本地数据库可以填写：localhost或者127.0.0.1

4、填写数据库开放的端口号，一般没设置的话默认都是3306

5、填写你需要连接的数据库名

6、填写数据库的用户名

7、填写数据库密码

8、这里会有一个驱动需要点击下载，图中是已经下载好了

9、填写自己的数据库连接url，然后可以点击9所在按钮进行测试连接，本地连接失败检查是否开启了mysql服务

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E6%9F%A5%E7%9C%8B%E6%95%B0%E6%8D%AE%E5%BA%93.png)

连接好了如上图所示，可以看到自己的数据库和表。



## 使用Persistence创建实体

idea 有个Tool window 叫作Persistence，可以将数据库表生成实体类：

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E6%89%93%E5%BC%80Persistence.png)



### 使用Persistence

要使用Persistence窗口需要在modules添加JPA模块

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E6%B7%BB%E5%8A%A0JPA.png)

### 查看Persistence

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E6%9F%A5%E7%9C%8BPersistence.png)

### 点击创建实体

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E7%82%B9%E5%87%BB%E4%BD%BF%E7%94%A8Pe.png)

### 选择需要创建实体的表

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E5%88%9B%E5%BB%BA%E5%AE%9E%E4%BD%93%E7%B1%BB.png)



## 配置JPA

如果module没有JPA选择，需要在Plugins添加上

![](https://raw.githubusercontent.com/yang-zhijiang/learn-data/master/idea%E4%BD%BF%E7%94%A8/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90%E5%AE%9E%E4%BD%93%E7%B1%BB/%E9%85%8D%E7%BD%AEJPA.jpg)

