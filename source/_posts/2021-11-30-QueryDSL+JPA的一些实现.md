---
title:      "QueryDSL+JPA的一些实现"
date:       2021-11-30
author:     "Planeter"
tags:
     - 20级
     - Java

---

## QueryDSL+JPA的一些实现

### JPA的一些问题

1. 不便于多表联结查询

   在JPA中,多表之间的关系，被映射为实体类之间的关系(OneToOne,OneToMany,ManyToMany等)。如果两个实体类之间没有声明关系与外键，你就不能把两个表联结起来查询。这种约束带来了很多麻烦，因为时候并不需要显式定义两个表之间的外键就能join。如果遵循这种约束，会使得外键增多，表之间的关系变得复杂。

2. JPQL没有SQL强大

   JPA将SQL封装起来，用面向对象的思想重新创造一个新的查询语言JPQL。但JPQL相比SQL更复杂，性能更低，且当用对象的概念取代表的概念后，由于受到框架的种种约束，查询灵活性更差了。

### QueryDSL简介

QueryDSL是基于各种ORM框架以及SQL之上的一个通用的查询框架。QueryDSL将原生的SQL编写方式转移到了Java程序内，内置了几乎所有的原生SQL的函数、关键字、语法等。我们可以通过 QueryDSL 提供的API 构造面向对象的查询语句，而不必将查询编写为拼接的字符串或将其写在XML文件中。

### 使用QueryDSL配合JPA进行查询

##### Maven配置QuetyDSL

QueryDSL 相关依赖 

```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <scope>provided</scope>
</dependency>
```

QueryDSL插件

该插件会扫描项目内注解了@Entity的实体类，并通过JPAAnnotationProcessor自动创建Q[实体类名]的查询实体。

```xml
<plugin>
    <groupId>com.mysema.maven</groupId>
    <artifactId>apt-maven-plugin</artifactId>
    <version>1.1.3</version>
    <executions>
        <execution>
            <goals>
                <goal>process</goal>
            </goals>
            <configuration>
                <outputDirectory>target/generated-sources/java</outputDirectory>
                <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### 

##### 实体类

**用户**

```java
@Data
@Entity
@Table(name = "user")
public class UserEntity implements Serializable
{
    @Id
    @Column(name = "id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(unique = true, nullable = false)
    private String username;
    private String password;
    private int age;
    private int gender;
    private String phone;
}
```

**订单**

```java
@Data
@Entity
@Table(name = "order")
public class OrderEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String address;
    private Date deliverTime;
    private String phone;
    private Long buyerId;
    private boolean complete;
}
```

##### DTO

```java
@Data
public class OrderDTO implements Serializable
{
    private String username;
    private int age;
    private int gender;
    private String phone;
    private String address;
    private Date deliverTime;
    private boolean complete;
}
```



##### 控制器

```java
@RestController
public class UserController
{
    @Resource
    private EntityManager entityManager;

    //查询工厂实例
    private JPAQueryFactory queryFactory;

    //实例化JPAQueryFactory
    @PostConstruct
    public void initFactory()
    {
        queryFactory = new JPAQueryFactory(entityManager);
    }
}
```

##### 常用查询

1. 多表联查

   ```java
   public List<User> selectOrderIdsByuserId(Long userId){
           //订单查询实体
           QOrder qOrder = QOrder.qOrder;
           //用户查询实体
           QUser qUser = QUser.qUser;
           return queryFactory
                   .select(qOrder)
                   .from(qOrder,qUser)
                   .where(
                           //两个表关联条件
                           qOrder.buyerId.eq(qUser.id)
                           .and(
                                   //查询指定userId的订单
                                   qUser.id.eq(userId)
                           )
                   )
                   //执行查询
                   .fetch();
   }
   ```

2. 查询实体并将结果封装至DTO

   ```java
   public List<OrderDTO> selectAllOrderDTO() {
       //用户查询实体
       QUser qUser = QUser.qUser;
       //订单查询实体
       QOrder qOrder = QOrder.qOrder;
       //连表查询实体并将结果封装至DTO
       return queryFactory
               .select(
                       Projections.bean(OrderDTO.class,
                                        qUser.username,
                                        qUser.age,
                                        qUser.gender,
                                        qUser.phone，
                                        qOrder.address,
                                        qOrder.deliveryTime,
                                        qOrder.complete)
               )
               .from(qUser，qOrder)
               .fetch();
   }
   ```

   Projections是QueryDSL中处理自定义返回结果集的对象，bean方法返回QBean。

   该解决方案相较于JPQL new对象显而更加灵活易用。

3. 动态查询

   ```java
   public List<User> selectUser(String username, String password, int gender, String phone) {
       //用户查询实体
       QUser qUser = QUser.qUser;
       //初始化组装条件(类似where 1=1)
       Predicate predicate = user.isNotNull().or(user.isNull());
       //执行动态条件拼装
       predicate = username == null ? predicate : ExpressionUtils.and(predicate, user.username.eq(username));
       predicate = password == null ? predicate : ExpressionUtils.and(predicate, user.password.eq(password));
       predicate = gender == null ? predicate : ExpressionUtils.and(predicate, user.phone.eq(gender));
       predicate = phone == null ? predicate : ExpressionUtils.and(predicate, user.phone.eq(phone));
       List<User> list = jpaQueryFactory
           .selectFrom(user)
           .where(predicate)               //执行条件
           .orderBy(user.id.asc())
           .fetch();
   }
   ```

4. 分页查询

   ```java
   public QueryResults<User> findAllUser(Pageable pageable) {
           QUser quser = QUser.user;
           return queryFactory
                   .selectFrom(quser)
                   .orderBy(
                           quser.id.asc()
                   )
                   .offset(pageable.getOffset())   //起始页
                   .limit(pageable.getPageSize())  //每页大小
                   .fetchResults();    //查询结果，包含实体集合与分页的信息，
   }
   ```

   

