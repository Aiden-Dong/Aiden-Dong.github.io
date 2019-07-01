---
layout:     post
title:      Mybatis | Mybatis 配置与初始化
subtitle:   
date:       2018-08-25
author:     Aiden
header-img: img/java.jpg
catalog: true 			
tags:								
    - Java
---

#### 1. 说明：


MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。

什么意思，看下例子,原生的jdbc案例:

```
//STEP 1. Import required packages
import java.sql.*;

public class FirstExample {
   // JDBC driver name and database URL
   static final String JDBC_DRIVER = "com.mysql.jdbc.Driver";  
   static final String DB_URL = "jdbc:mysql://localhost/emp";

   //  Database credentials
   static final String USER = "root";
   static final String PASS = "123456";

   public static void main(String[] args) {
   Connection conn = null;
   Statement stmt = null;
   try{
      //STEP 2: Register JDBC driver
      Class.forName("com.mysql.jdbc.Driver");

      //STEP 3: Open a connection
      System.out.println("Connecting to database...");
      conn = DriverManager.getConnection(DB_URL,USER,PASS);

      //STEP 4: Execute a query
      System.out.println("Creating statement...");
      stmt = conn.createStatement();
      String sql;
      sql = "SELECT id, first, last, age FROM Employees";
      ResultSet rs = stmt.executeQuery(sql);

      //STEP 5: Extract data from result set
      while(rs.next()){
         //Retrieve by column name
         int id  = rs.getInt("id");
         int age = rs.getInt("age");
         String first = rs.getString("first");
         String last = rs.getString("last");
      }
      //STEP 6: Clean-up environment
      rs.close();
      stmt.close();
      conn.close();
   }catch(SQLException se){
      //Handle errors for JDBC
      se.printStackTrace();
   }catch(Exception e){
      //Handle errors for Class.forName
      e.printStackTrace();
   }finally{
      //finally block used to close resources
      try{
         if(stmt!=null)
            stmt.close();
      }catch(SQLException se2){
      }// nothing we can do
      try{
         if(conn!=null)
            conn.close();
      }catch(SQLException se){
         se.printStackTrace();
      }//end finally try
   }//end try
   System.out.println("There are so thing wrong!");
}//end main
}//end FirstExample
```

原生的jdbc 代码阅读困难， 耦合性强，复用能力查，设计模式粗糙。为了将这个过程格式化或者说是模板话，便出来了好多工具。

例如:**[DBUtil](https://commons.apache.org/proper/commons-dbutils/examples.html)**, **[Mybaitis](http://www.mybatis.org/mybatis-3/zh/index.html)**, **[Hibernate](http://hibernate.org/)** 等

接下来我们就介绍下Mybatis的使用方式。

#### 1. 项目依赖:

```
<!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.4.1</version>
</dependency>
```


#### 2. 引用：

使用Mybaits 分为两步:

1. 获取 `SqlSessionFactory`
2. 获取 `SqlSession`
3. 获取数据、执行更新。

简单吧。

##### 2.1 从xml中获取 SqlSessionFaction:

```
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```
这里需要解释下 `mybatis-config.xml`

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

  <environments default="development">
    <environment id="development">

      <transactionManager type="JDBC"/>

      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>


  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
</configuration>
```
- **environments** :
表示环境配置。
在实际开发中，我们通常有多个环境，例如开发环境(development)、测试环境(test)、生产环境(production)等，不同的环境的配置都是不同的。
因此在environments元素中，可以配置多个表示具体某个环境的environment子元素。
而default属性用于指定默认的环境。

- **transactionManager**:
事务管理器，属性type取值有2个，JDBC|MANAGED。
其中：JDBC表示任何对数据库的修改操作，mybatis都会自动开启事务。
这里配置的是MANAGED，表示事务由应用程序来控制，也就是我们需要手动的开启事务和提交事务。
和spring整合时，开启和提交事务的操作交由spring来管理。

- **datasource** :
表示数据源配置。这个更好理解，因为不同的环境中，我们访问的数据库url、username、password都是不同的，因此在每个environment元素下面都有一个dataSource配置。
属性type表示使用的数据源类型，取值有三个：UNPOOLED|POOLED|JNDI，
    这里指定POOLED，表示使用mybatis自带的PooledDataSource。
    而dataSource内部通过property元素配置的属性，都是PooledDataSource支持的。
    注意不同的数据源实现，可以配置的property是不同的。

- **mappers** :
表示映射文件列表，前面提到通常我们针对数据库中每张表，都会建立一个映射文件。
而在mappers元素中，就通过mapper元素，列出了所有配置文件的路径。例如mapper元素可以通过以下属性指定映射文件的路径：
   resource属性：表示映射文件位于classpath下。例如上面的配置中就表示在classpath的mappers目录下，有一个UserMapper.xml映射文件
   url属性：使用完全限定资源定位符指定映射文件路径，如file:///var/mappers/uthorMapper.xml
   class属性：通过java类来配置映射关系，可以一个java映射类对应一个xml映射文件
   package：如果有多个java映射类，且位于同一个包下面，我们可以直接使用package属性指定包名，不需要为每个java映射配置一个class属性。

> **如何引用第三方 Datasource** :

开源社区有不少优秀的DataSource 我们可以使用， 解锁的正确方式是：

```
<dataSource type="com.test.mysite.common.MyselfDefineDataSourceFactory">
    <property name="driverClass" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/site-aliyun"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</dataSource>
```

```
package com.test.mysite.common;

import org.apache.ibatis.datasource.unpooled.UnpooledDataSourceFactory;

import com.alibaba.druid.pool.DruidDataSource;

public class MyselfDefineDataSourceFactory extends UnpooledDataSourceFactory {
    public MyselfDefineDataSourceFactory() {
        this.dataSource = new DruidDataSource();
    }
}
```

##### 2.2 在代码中构建SqlSessionFaction:

```
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();        // 获取 DataSource
TransactionFactory transactionFactory = new JdbcTransactionFactory();     // jdbc transaction
Environment environment = new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

这种方式也是简单快捷.

##### 获取 SqlSession 的方式：

当拿到 `SqlSessionFactory` 之后，便可以拿到 **SqlSession**,每次使用完之后必须关闭:

```
SqlSession session = sqlSessionFactory.openSession();
try {
  Blog blog = (Blog) session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
} finally {
  session.close();
}
```

每次都要 close , 代码看上去不美观，作者也很烦呀， 看到在mybatis-spring中有一个使用代理实现自动close的 SqlSession 子类**SqlSessionTemplate**解决了这一问题，但是在纯种mybatis中，还是没有较好的解决办法，继续烦恼中(mybatis拿到的session实体为:**DefaultSqlSession**)

#### 3. 实例

在文章最后贴出作者封装的 SqlSessionFactory 构造类:

配置信息:

```
# 数据库连接池
datasource {
  driver : "org.h2.Driver",
  url : "jdbc:h2:mem:testDB",
  userName : "test",
  passWord : "test",
  connect-init-size : 5,
  connect-min-size : 2,
  connect-max-size : 10
}
```

封装类:

```
package com.mobvista.dataplatform.cluster.common;

import com.mobvista.dataplatform.cluster.dao.UserDefineActionMapper;
import com.mobvista.dataplatform.cluster.dao.cluster.ClusterMessageMapper;
import com.mobvista.dataplatform.cluster.dao.cluster.InstanceMessageMapper;
import com.mobvista.dataplatform.cluster.dao.cluster.InstanceStrategyMapper;
import com.mobvista.dataplatform.cluster.dao.security.SecurityMessageMapper;
import com.typesafe.config.Config;
import org.apache.commons.dbcp.BasicDataSource;
import org.apache.ibatis.mapping.Environment;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.apache.ibatis.transaction.TransactionFactory;
import org.apache.ibatis.transaction.jdbc.JdbcTransactionFactory;

import javax.sql.DataSource;

/***
 * <pre>
 *     数据库查询工具
 * </pre>
 * @user : saligia
 * @date : 2018-08-23
 */
public class QueryManager {

    private static final String DATASOURCE_DRIVER = "driver";
    private static final String DATASOURCE_URL = "url";
    private static final String DATASOURCE_USER = "userName";
    private static final String DATASOURCE_PASSWD = "passWord";

    private static final String DATASOURCE_INIT_SIZE = "connect-init-size";
    private static final String DATASOURCE_MIN_SIZE = "connect-min-size";
    private static final String DATASOURCE_MAX_SIZE = "connect-max-size";

    private static final String DEVELOPMENT = "development";

    private DataSource dataSource = null;
    private SqlSessionFactory sqlSessionFactory = null;

    private static QueryManager queryManager = null;

    private QueryManager(){
        throw new IllegalArgumentException("invalid data config error");
    }


    private QueryManager(Config dataConfig) {
        initDataSource(dataConfig);
        initSQLSessionFaction();
    }


    /**
     * <pre>
     *   初始化数据库连接池
     * </pre>
     *
     * @param dataConfig 数据库配置信息
     */
    private void initDataSource(Config dataConfig){

        BasicDataSource dataSource = new BasicDataSource();

        if (!dataConfig.hasPath(DATASOURCE_DRIVER)){
            throw new IllegalArgumentException("Init datasource error : driver invalid");
        }
        dataSource.setDriverClassName(dataConfig.getString(DATASOURCE_DRIVER));

        if(!dataConfig.hasPath(DATASOURCE_URL)){
            throw new IllegalArgumentException("Init datasource error : url invalid");
        }
        dataSource.setUrl(dataConfig.getString(DATASOURCE_URL));

        if(!dataConfig.hasPath(DATASOURCE_USER)){
            throw new IllegalArgumentException("Init datasource error : user invalid");
        }
        dataSource.setUsername(dataConfig.getString(DATASOURCE_USER));

        if(!dataConfig.hasPath(DATASOURCE_PASSWD)){
            throw new IllegalArgumentException("Init datasource error : passwd invalid");
        }
        dataSource.setPassword(dataConfig.getString(DATASOURCE_PASSWD));

        if(dataConfig.hasPath(DATASOURCE_INIT_SIZE)){
           dataSource.setInitialSize(dataConfig.getInt(DATASOURCE_INIT_SIZE));
        }

        if(dataConfig.hasPath(DATASOURCE_MIN_SIZE)){
            dataSource.setMinIdle(dataConfig.getInt(DATASOURCE_MIN_SIZE));
        }

        if(dataConfig.hasPath(DATASOURCE_MAX_SIZE)){
            dataSource.setMaxActive(dataConfig.getInt(DATASOURCE_MAX_SIZE));
        }

        this.dataSource = dataSource;
    }

    /**
     * <pre>
     *     初始化 SQLSession
     * </pre>
     *
     * @param
     */
    private void initSQLSessionFaction(){
        TransactionFactory transactionFactory = new JdbcTransactionFactory();
        Environment environment = new Environment(DEVELOPMENT, transactionFactory, dataSource);
        Configuration configuration = new Configuration(environment);
        configuration.setMapUnderscoreToCamelCase(true);

        configuration.addMapper(ClusterMessageMapper.class);
        configuration.addMapper(InstanceMessageMapper.class);
        configuration.addMapper(InstanceStrategyMapper.class);
        configuration.addMapper(SecurityMessageMapper.class);
        configuration.addMapper(UserDefineActionMapper.class);

        this.sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
    }

    /**
     * <pre>
     *     构建 QueryManager 过程
     * </pre>
     *
     * @param dataSource 数据库连接池配置信息
     */
    public static QueryManager createQueryManager(Config dataSource){
        if(queryManager == null){
            queryManager = new QueryManager(dataSource);
        }
        return  queryManager;
    }

    /**
     * <pre>
     *     获取 sql 执行session
     *     -----------------------
     *     使用结束后必须 close
     * </pre>
     *
     * @param
     */
    public SqlSession getSession(){
        return sqlSessionFactory.openSession(true);
    }


    public DataSource getDataSource(){
        return this.dataSource;
    }

}
```


---

另附 ：

数据库连接池：[数据库连接池原理详解与自定义连接池实现 - CSDN博客](https://blog.csdn.net/fuyuwei2015/article/details/72419975)
