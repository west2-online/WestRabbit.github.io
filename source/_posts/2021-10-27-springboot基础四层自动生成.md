## # web开发框架renren

2021-10-28

#### 一、renren-fast

renren-fast是一个轻量级的Spring Boot2.1快速开发平台，其设计目标是开发迅速、学习简单、轻量级、易扩展；使用Spring Boot、Shiro、MyBatis、Redis、Bootstrap、Vue2.x等框架，包含：管理员列表、角色管理、菜单管理、定时任务、参数管理、代码生成器、日志管理、云存储、API模块(APP接口开发利器)、前后端分离等。

- [renren-fast]: https://gitee.com/renrenio/renren-fast

- [官方文档]: https://www.renren.io/guide

### 项目特点

- 友好的代码结构及注释，便于阅读及二次开发
- 实现前后端分离，通过token进行数据交互，前端再也不用关注后端技术
- 灵活的权限控制，可控制到页面或按钮，满足绝大部分的权限需求
- 页面交互使用Vue2.x，极大的提高了开发效率
- 完善的代码生成机制，可在线生成entity、xml、dao、service、vue、sql代码，减少70%以上的开发任务
- 引入quartz定时任务，可动态完成任务的添加、修改、删除、暂停、恢复及日志查看等功能
- 引入API模板，根据token作为登录令牌，极大的方便了APP接口开发
- 引入Hibernate Validator校验框架，轻松实现后端校验
- 引入云存储服务，已支持：七牛云、阿里云、腾讯云等
- 引入swagger文档支持，方便编写API接口文档

### 使用教程

1. 从git上clone源码
2. 在ide中安装lombok插件 代码中会有@Data注释 不安装会提示找不到getter setter方法
3. 创建数据库renren_fast
4. 修改application.yml，修改MySql账号和密码
5. 启动启动类

##### 注：若使用的框架版本不一致框架使用不统一（如项目使用spring security而renren-fast使用shiro）可自行调整

## 二、renren-generator

Renren-generator代码生成的思想主要是通过volocity模板并打成zip包的形式。

它的技术栈主要如下:

- 核心框架：Spring Boot 2.0
- 安全框架：Apache Shiro 1.4
- 视图框架：Spring MVC 5.0
- 持久层框架：Mybatis 3.3
- 定时器：Quatz 2.3
- 数据库连接池：Druid 1.1
- 日志管理：SLF4J 1.7 、Log4j
- 页面交互： Vue 2.x

#### 使用教程

[git地址]: https://gitee.com/mfj3657648/renren-generator

1.通过git下载源码

2.修改application.yml ， 修改MySql账号和密码、数据库名称

3.运行启动类

4.访问项目路径http://localhost:[port]/renren-generator

5.选择对应的表，生成对应的压缩包

6.将压缩包解压后复制到项目相应的目录下

renren-generator会自动生成dao，service，controller，entity四个模块且controller包内会自动生成增删改查方法 不需要手动编写，为项目的开头极大的节省了时间。

##### 注：renren-generator里会用到一些工具类 比如RRException自定义异常类或结果封装类等 这些类可以在renren-fast中找到 也可以自定义重写



有了这两个工具的帮助 是不是觉得万事开头也没有那么难了呢

mARk-LzZ

 





