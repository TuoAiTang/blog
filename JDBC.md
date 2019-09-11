# JDBC

Connecttion
DriverManager
Driver
Statement
ResultSet
PrepamentStatement
PSCache

执行 Class.forName(driverClass) 时会执行这个static registry
```java
package com.mysql.jdbc;
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    //
    // Register ourselves with the DriverManager
    //
    static {
        try {
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }

    /**
     * Construct a new driver and register it with DriverManager
     * 
     * @throws SQLException
     *             if a database error occurs.
     */
    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
```

## 预编译

预编译在其他数据库中性能提升蛮大的，不过在mysql中则不一定。
 
在使用ps的时候，如果没有显示的声明启用服务端预编译，那么就默认启用本地预编译。事实上本地预编译与服务端预编译，在最新版的mysql驱动上，差距并不大。
 
但是，无论是本地预编译，还是服务端预编译，在执行SQL的时候，每一次都会向MySQL服务器进行预编译的请求，然后执行，释放。并且增加了 RTT 。 所以最后结果，总是会比s慢。
 
总之，要想在mysql中通过预编译提升sql的执行效率，必须在建立连接的时候，开启服务端预编译与缓存。
 
ps的好处，除了效率提升，最重要的就是可以防止SQL注入。


## try-catch
```java
try{
	try{

	}
}catch(Exception e){
	e.printStackTrace();
}
```

多层嵌套对性能的影响？

两个try{}可以只有一个catch

## try-with
```java
try(Connection connection = ConnectAndRelease.getConnection()){
	connection.(..)
}catch(Exception e){
	e.printStackTrace();
}
```

## mysql支持游标吗?
> Druid配置文档中：是否缓存preparedStatement，也就是PSCache。PSCache对支持游标的数据库性能提升巨大，比如说oracle。在mysql下建议关闭。

## JpaRepository
在自增id的情况下：
执行save()返回一个对象，如果它在插入期间没有被改变，则返回原对象，否则创建新的对象返回。

值得注意：JPA 新增和修改用的都是save. 它根据实体类的id是否为0来判断是进行增加还是修改

## 数据传递层次

mysql server --> dataSource --> JPARepository

## Hibernate

创建hibernate.properties放在resources目录下会被自动加载用来配置。

hibernate.hbm2ddl.auto = validate | update | create | create-drop

update ： 在实体类的属性上的注解变更时不会更新，在类上的@Table注解会更新。

增添类属性会映射更新表结构。

hibernate创建表默认MyISAM引擎，需要增加配置：hibernate.dialect.storage_engine=innodb
## DML
DML（data manipulation language）数据操纵语言：

就是我们最经常用到的 SELECT、UPDATE、INSERT、DELETE。 主要用来对数据库的数据进行一些操作。

## DDL
DDL（data definition language）数据库定义语言：

其实就是我们在创建表的时候用到的一些sql，比如说：CREATE、ALTER、DROP等。DDL主要是用在定义或改变表的结构，数据类型，表之间的链接和约束等初始化工作上。

## 通过在@Modifying + @Query里执行语句，来达到自定义更新会报异常

1. InvalidDataAccessApiUsageException
2. TransactionRequiredException

解决：在service 层 UserService中, 需要调用repository层UserRepository中需要事务的方法时，为这个方法加上@Transactional注解。

## JDBC template
提供了查询的一些模板，不用手动去管理连接。不用处理复杂的异常。基于回调函数的方式实现对结果集的提取。
模板的实现方式大多都是基于回调函数实现的，定义时只需传入一个实现了某个接口的匿名类。模板中定义一些接口，操作这些接口来做通用的逻辑执行。

## ORM 和 JDBC/JDBC template

场景1：需要查询满足某条件的记录的某几个字段。

视图可以作为解决的一个手段。

假设有student(id int, name varchar, age int)表。
问题：
select * from student 和 select id, name from student 空间时间对比？

场景2：一个表的字段特别多，这个时候还用orm吗？


问题：orm 和 jdbc 性能？ 监控 ？

现在的各种ORM框架都在尝试使用各种方法来减轻这块（**LazyLoad**，**Cache**），效果还是很显著的。

场景3： 复杂查询？

视图可以作为解决的一个手段。

## JDBC 开启事务

