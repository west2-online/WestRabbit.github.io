---
title:      "MybatisPlus -- 一个Mybatis的增强框架"
date:       2020-11-28
author:     "Forestj"
tags:
     - 18级
     - Java
---

# MybatisPlus -- 一个Mybatis的增强框架

在使用Mybatis时，所有的查询sql语句都需要程序员手动来编写，其中，单表的，最基础的sql语句更是占据了其中很大一部分，这些语句没有太多的技术含量，却要消耗大量宝贵的时间。于是，MybatisPlus出现了，借助MybatisPlus，我们可以自动生成大量的单表sql语句，提升开发效率。

## 前置技能

- SpringBoot

- Mybatis

## 简介

```
MyBatis-Plus （简称 MP）是一个 MyBatis 的增强工具，在 MyBatis 的基础上只做增强不做改变，为简化开发、提高效率而生。
																--来自MyBatis-Plus官方文档
```

## 快速上手

数据库准备

```mysql
CREATE DATABASE mpstudy;
USE mpstudy;

DROP TABLE IF EXISTS user;
CREATE TABLE `user`
(
	id int(11) NOT NULL COMMENT '主键ID',
	`name` VARCHAR(30) NULL DEFAULT NULL COMMENT '姓名',
	age INT(11) NULL DEFAULT NULL COMMENT '年龄',
	email VARCHAR(50) NULL DEFAULT NULL COMMENT '邮箱',
	PRIMARY KEY (id)
);

DELETE FROM user;
INSERT INTO user (id, name, age, email) VALUES
(1, 'Jone', 18, 'test1@baomidou.com'),
(2, 'Jack', 20, 'test2@baomidou.com'),
(3, 'Tom', 28, 'test3@baomidou.com'),
(4, 'Sandy', 21, 'test4@baomidou.com'),
(5, 'Billie', 24, 'test5@baomidou.com');

```

*数据库参考自官方文档*



新建一个SpringBoot工程，并引入MyBatis-Plus的依赖

```xml
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.4.1</version>
    </dependency>
```

- 引入MyBatis-Plus依赖之后最好不要重复引入Mybatis依赖



项目配置

```yaml
server:
  port: 12345

spring:
  datasource:
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/mpstudy?useSSL=false&serverTimezone=CTT&useUnicode=true&characterEncoding=UTF-8

mybatis-plus: # 这里是mybatis-plus，而不是mybatis
  mapper-locations: classpath:mapperxml/*.xml
  configuration:
    map-underscore-to-camel-case: true # 开启驼峰命名
```



项目基础结构搭建

- **实体类**

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@TableName(value = "`user`")
public class User {
    // 主键ID
    @TableId(value = "id")
    private Integer id;
    
     //姓名
    @TableField(value = "`name`")
    private String name;
    
     //年龄
    @TableField(value = "age")
    private Integer age;
    
     //邮箱
    @TableField(value = "email")
    private String email;
}
```

说明：

- @TableName中可以标识数据库的表名，当表名和实体类名不一致(不能自动转换)时必须配置
- @TableId标识数据库主键，value值为数据库列名，虽然大部分时候MybatisPlus可以自动识别主键，但是偶尔会有问题，这个注解一般都要配
- @TableField标识非主键列，value值为数据库列名,一致(或可以转换)时可以省略



- **mapper**

```java
@Mapper
@Repository
public interface UserMapper extends BaseMapper<User> {
}
```

mapper类的主要变化就是继承了BaseMapper<T>这个类，一旦继承，就会自动实现BaseMapper中定义好的各种方法，不用我们来写sql了



- 写个方法测试一下

  controller

  ```java
      @Autowired
      UserService userService;
  
      @GetMapping("/user/{id}")
      public User getUser(@PathVariable(value = "id") Integer id){
          User user = userService.findUserById(id);
          return user;
      }
  ```

  service

  ```java
      @Autowired
      UserMapper userMapper;
      
      @Override
      public User findUserById(Integer id) {
          return userMapper.selectById(id);
      }
  ```

  完成，运行一下看看效果
  
  ```bash
  $ curl http://192.168.1.6:12345/user/1
  {"id":1,"name":"Jone","age":18,"email":"test1@baomidou.com"}
  ```
  
  

除了上述的select外，BaseMapper还提供了其他大量的CRUD方法，各个方法基本见名知意，例如deleteById, selectList等等，具体可以查看BaseMapper类的结构



至此，通过BaseMapper，我们已经实现了最简单的CRUD。

## service层接口

很多时候，我们的Service层承担的功能比较少，比如上文的查询用户，service层只调用了mapper层并返回数据，对于这一类简单的查询，我们是否可以更简化呢？

答案是可以的，借助MybatisPlus提供的service层接口。

修改我们的service层



```java
public interface UserService extends IService<User>{
    
}
```

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService{

}
```

让service层的接口继承IService<T>,让实现类继承ServiceImpl<U, V>,这样，就成功引入了service层的CRUD接口，在IService中，同样有着大量的例如save(),update(),getOne()等方法，具体可以查看IService接口的定义，他们都已经被自动实现了

现在，我们在controller里就可以直接通过service调用这些方法。

```java
    @GetMapping("user/{id}")
    public User getUser(@PathVariable(value = "id") Integer id){
        return userService.getById(id);
    }
```

效果和之前完全相同

## 条件查询

上述所演示的查询都是最简单的查询，在真实业务中，绝大部分的查询都需要拼接条件，这时候，可以使用MybatisPlus提供的Wrapper类来进行条件的拼接。

这里假设我们要查询年龄>x的用户,我们可以这么写

```java
    public List<User> findUserAgeGt(Integer age) {
        // gt表示某个列的值大于参数值
        QueryWrapper<User> wrapper = new QueryWrapper<User>().gt("age", age);
        return userMapper.selectList(wrapper);
    }
```

首先，代码中定义了QueryWrapper<T>(更新时可以使用UpdateWrapper)，在这个类中可以通过链式的方式来拼接条件语句。除了where后面的条件语句，还可以拼接group by， order by, having等语句，具体的各类方法可以到官方文档进行查看，这里就不详细列出了。文档地址：https://baomidou.com/guide/wrapper.html#abstractwrapper

- 查询部分字段

  很多时候我们并不会去查询一整张表，可能只会查询其中的部分字段，通过QueryWrapper的select方法，我们可以实现只查询部分字段。

  select()方法的签名

```java
public QueryWrapper<T> select(String... columns)
```

可以看到，select方法可以接收不固定数量的String参数，这些参数就是要查询的列名。



以查询所有的用户姓名为例，在Service中，

```java
    public List<String> findName() {
        //select方法传了一个"name",代表只查询name这一列
        QueryWrapper<User> wrapper = new QueryWrapper<User>().select("name");
        List<User> users = userMapper.selectList(wrapper);
        //通过java8的流特性将查询到的数据映射为Name
        return users.stream().map(User::getName).collect(Collectors.toList());
    }
```

运行结果

```java
[Jone, Jack, Tom, Sandy, Billie]
```

这样，我们就做到了只查询部分列，当然，在QueryWrapper中也可以拼接其他的查询条件。



## 分页查询

当查询到的数据条目非常多时，我们有必要对数据进行分页，减少一次传输的数据量，在MybatisPlus中，有内置的分页插件，可以简单的帮助我们完成分页操作。

首先，需要进行分页参数的配置(来自官方文档)

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        PaginationInterceptor paginationInterceptor = new PaginationInterceptor();
        // 设置请求的页面大于最大页后操作， true调回到首页，false 继续请求  默认false
        // paginationInterceptor.setOverflow(false);
        // 设置最大单页限制数量，默认 500 条，-1 不受限制
        // paginationInterceptor.setLimit(500);
        // 开启 count 的 join 优化,只针对部分 left join
        paginationInterceptor.setCountSqlParser(new JsqlParserCountOptimize(true));
        return paginationInterceptor;
    }
}
```

配置完成以后，就可以直接使用了

以分页查询所有用户为例

```java
    public IPage<User> findByPage(int pageSize, int pageNum) {
    	//pageSize为一页的大小，pageNum为当前页的页码(从1开始)
        Page<User> userList = userMapper.selectPage(new Page<>(pageNum, pageSize), null);
        return userList;
    }
```

我们构造了一个Page对象，并传入selectPage方法，这样，MybatisPlus就会自动帮我们处理好分页查询

再来看看返回的Page的部分常用数据结构

```java
public class Page<T> implements IPage<T> {

    /**
     * 查询数据列表
     */
    protected List<T> records = Collections.emptyList();

    /**
     * 总数
     */
    protected long total = 0;
    /**
     * 每页显示条数，默认 10
     */
    protected long size = 10;

    /**
     * 当前页
     */
    protected long current = 1;
    ...
   }
```

以上就是分页查询会返回的数据结构，查询完成后就可以直接获得这些数据。



- 注解写sql并自动注入page

对于一些复杂的查询接口，我们可能会通过mapper或注解方式手写sql，在这些地方，我们也可以通过传入Page让MybatisPlus帮我们自动拼接分页参数

下面以分页查询用户为例

```java
public interface UserMapper extends BaseMapper<User> {
    @Select("select * from user")
    Page<User> findUserByPage(Page page);
}
```

在UserMapper中添加如下方法，注意要传一个Page参数且返回类型也为Page.

调用该方法，查看日志

```
==>  Preparing: SELECT id,`name`,age,email FROM `user` WHERE id=?
==> Parameters: 1(Integer)
<==    Columns: id, name, age, email
<==        Row: 1, Jone, 18, test1@baomidou.com
<==      Total: 1
```

可以发现这个方法被自动拼接了分页语句，通过这种方式，我们同样可以实现分页。



至此，MybatisPlus的一些基础功能已经介绍完毕，想更深入的了解可以去查看官方文档。
