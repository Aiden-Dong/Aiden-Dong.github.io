---
layout:     post
title:      Mybatis | Mybatis 注解开发方式
subtitle:   
date:       2018-08-26
author:     Aiden
header-img: img/java.jpg
catalog: true 			
tags:								
    - Java
---

### 1. 背景

mybatis最初配置信息是基于 XML ,映射语句(SQL)也是定义在 XML 中的。而到了 MyBatis 3提供了新的基于注解的配置。

这里讲述 注解开发方式:

首先我们需要获取 **SqlSession**:

```
SqlSession session = sqlSessionFactory.openSession(true);
```

参数设置为`true`表示开启自动提交模式。


session 在注解形式的使用方式如:

```
ClusterMessage clusterMessage = new ClusterMessage();
clusterMessage.setClusterId(1);
clusterMessage.setClusterName("useName");
clusterMessage.setClusterTime(new Date().getTime());
clusterMessage.setClusterAddress("localhost");
clusterMessage.setClusterAccessUser("user");
clusterMessage.setClusterAccessPasswd("test");

session.getMapper(ClusterMessageMapper.class).updateClusterMessage(clusterMessage);
```

所以mybatis 的使用使用三部分:

1. 数据查询主体 : `SqlSession`
2. 查询映射层 : `Mapper接口`
3. 数据维护层 : `Bean 设计`


### 2. 案例

#### table :

```
CREATE TABLE cluster_manager(
    cluster_id int PRIMARY KEY AUTO_INCREMENT ,
    cluster_name VARCHAR(20) NOT NULL,
    cluster_time BIGINT NOT NULL ,
    cluster_address VARCHAR(50) NOT NULL,
    cluster_access_user VARCHAR(50) NOT NULL ,
    cluster_access_passwd VARCHAR(50) NOT NULL
)AUTO_INCREMENT=0;
```

#### Bean :

```
public class ClusterMessage {
    int clusterId = 0;                     // 集群ID
    String clusterName = "";               // 集群名称
    Long clusterTime = 0l;                 // 集群上线时间
    String clusterAddress = "";            // 集群内网访问地址
    String clusterAccessUser = "";         // 集群访问用户
    String clusterAccessPasswd = "";       // 集群访问密码

    public int getClusterId() {
        return clusterId;
    }
    public void setClusterId(int clusterId) {
        this.clusterId = clusterId;
    }
    public String getClusterName() {
        return clusterName;
    }
    public void setClusterName(String clusterName) {
        this.clusterName = clusterName;
    }
    public Long getClusterTime() {
        return clusterTime;
    }
    public void setClusterTime(Long clusterTime) {
        this.clusterTime = clusterTime;
    }
    public String getClusterAddress() {
        return clusterAddress;
    }
    public void setClusterAddress(String clusterAddress) {
        this.clusterAddress = clusterAddress;
    }
    public String getClusterAccessUser() {
        return clusterAccessUser;
    }
    public void setClusterAccessUser(String clusterAccessUser) {
        this.clusterAccessUser = clusterAccessUser;
    }
    public String getClusterAccessPasswd() {
        return clusterAccessPasswd;
    }
    public void setClusterAccessPasswd(String clusterAccessPasswd) {
        this.clusterAccessPasswd = clusterAccessPasswd;
    }

    @Override
    public String toString() {
        return "ClusterMessage{" +
                "clusterId='" + clusterId + '\'' +
                ", clusterName='" + clusterName + '\'' +
                ", clusterTime=" + clusterTime +
                ", clusterAddress='" + clusterAddress + '\'' +
                ", clusterAccessUser='" + clusterAccessUser + '\'' +
                ", clusterAccessPasswd='" + clusterAccessPasswd + '\'' +
                '}';
    }
}
```

#### Mapper:

```
public interface ClusterMessageMapper {

    // Insert
    @Insert("insert into cluster_manager(cluster_name, cluster_time, cluster_address, cluster_access_user, cluster_access_passwd) " +
            "values(#{clusterName}, #{clusterTime}, #{clusterAddress}, #{clusterAccessUser}, #{clusterAccessPasswd})")
    @Options(useGeneratedKeys = true, keyColumn = "cluster_id", keyProperty = "clusterId")
    public void insertClusterMessage(ClusterMessage clusterMessage);

    // select
    @Select("select * from cluster_manager")
    @Results(
            id = "clusterMessage",
            value = {
                    @Result(column = "cluster_name", property = "clusterName", javaType = String.class, jdbcType = JdbcType.VARCHAR),
                    @Result(column = "cluster_time", property = "clusterTime", javaType = Long.class, jdbcType = JdbcType.BIGINT),
                    @Result(column = "cluster_address", property = "clusterAddress", javaType = String.class, jdbcType = JdbcType.VARCHAR),
                    @Result(column = "cluster_access_user", property = "clusterAccessUser", javaType = String.class, jdbcType = JdbcType.VARCHAR),
                    @Result(column = "cluster_access_passwd", property = "clusterAccessPasswd", javaType = String.class, jdbcType = JdbcType.VARCHAR)
            }
    )
    public List<ClusterMessage> getClusterMessage();

    @Select("select * from cluster_manager")
    @MapKey("clusterId")
    public Map<Integer, ClusterMessage> getClusterMessageMapper();


    @Select("select * from cluster_manager where cluster_id=#{clusterId}")
    @ResultMap("clusterMessage")
    public ClusterMessage getClusterMessageById(@Param("clusterId") int clusterId);


    // update
    @Update("update cluster_manager set cluster_name=#{clusterName} where cluster_id=#{clusterId}")
    public void updateClusterMessage(ClusterMessage clusterMessage);

    // delete
    @Delete("delete from cluster_manager where cluster_id=#{clusterId}")
    public void deleteClusterMessage(@Param("clusterId")int clusterId);
}
```

### 3. 详解:

这里主要讲解 Mapper 层的开发规则。

sql 类型主要分成 : select`@Select(${sql})`, update`@Update(${sql})`, insert`@Insert($sql)`, delete`(${sql})`.

#### @Select:

**@Results** 用来设置table信息与bean相关字段的映射关系， 每一个字段的关系使用 **@Result**控制。

默认情况下对于每一table字段，例如`name`, 会调用 `bean` 中的 `setName(..)`.  如果找不到，对于新版本的 mybatis 会报错。
例如上面的 `cluster_name` 会调用 `setCluster_name()`. 但是java 中使用的 `clusterName`,可以通过 **Result** 注解控制.

**@ResultMap** 可以通过Id,应用其他的**Results**

还有一种更改映射的方式:

```
TransactionFactory transactionFactory = new JdbcTransactionFactory();
Environment environment = new Environment(DEVELOPMENT, transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.setMapUnderscoreToCamelCase(true);
```

**mapUnderscoreToCamelCase** 设置为true, 之后会自动实现 mysql 中的unix命名方式转为java的驼峰表示法。


**@MapKey** 此注解应用将查询数据转为 Map<>, 注意的是MapKey()中的id最终调用bean的getId 获取数据，所以需要映射bean字段而不是table.

#### @Param注解:

@Param注解用于给方法参数起一个名字。以下是笔者总结的使用原则：
在方法只接受一个参数的情况下，可以不使用@Param。
在方法接受多个参数的情况下，建议一定要使用@Param注解给参数命名。

#### @Insert :

insert 时获取自增主键的方式:

法一:

```
@Options(useGeneratedKeys = true, keyColumn = "cluster_id", keyProperty = "clusterId")
```

法二:

```
@SelectKey(statement="SELECT LAST_INSERT_ID()", keyProperty="clusterId", before=false, resultType=Integer.class)
```

### 4. #与$的区别:

`#{}` 的作用主要是替换预编译语句(PrepareStatement)中的占位符`?`:

对于 : `INSERT INTO user (name) VALUES (#{name});` ==> `INSERT INTO user (name) VALUES (?);`

`${}` 符号的作用是直接进行字符串替换:

对于 : `INSERT INTO user (name) VALUES ('${name}');` ==> `INSERT INTO user (name) VALUES ('tianshozhi');`
