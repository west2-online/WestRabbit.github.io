---
title: 关于Tomcat的配置等
date: 2016-11-23 21:03:34
author: "Csming"
catalog: true
tags:
     - 14级
     - Java
---

***
刚看javaweb的视频时记得笔记，关于Tomcat的配置和javaweb的简介
# JavaWeb是什么
* **JavaWeb是由一组Servlet、HTML页、类、以及其他可以被绑定的资源构成**
* **可以在各种供应商提供的实现Servlet规范的Servlet容器中运行**
JavaWeb包含了:Servlet、JSP、实用类、静态文档如HTML，图片等、提供描述Web应用的信息(web.xml)
***
#### 请求模式
* **在Servlet容器中编写Servlet和JSP**
![请求模式](http://i1.piimg.com/567571/7c19ece627715d50.png)
***
# Servlet容器
* **Servlet容器为JavaWeb应用提供运行时环境**
* **负责管理Servlet和JSP的生命周期**
* **管理他们的共享数据**
* **Servlet容器也称为JavaWeb应用容器或者Servlet/JSP容器**
* **Servlet容器有:Tomcat、Resin、J2EE等**
***
# Tomcat
* **Tomcat是一个免费的开源的Servlet容器**
* **由Apache、Sun等公司开发**
* **决定了Servlet和JSP的生命周期、提供生命周期方法等**
## 部署Tomcat
* **将Tomcat源码解压到某目录下如：D:\\apache-tomcat**
* **源文件**  
bin：存放启动和关闭Tomcat的脚本文件  
conf：存放Tomcat服务器的各种配置文件  
lib：存放Tomcat服务器和所有web应用程序需要访问的jar文件  
logs：日志文件  
temp：Tomcat运行时临时产生的文件  
webapps：当发布web应用时，通常把web应用程序的目录及文件放到这个目录下  
work：将JSP生成的Servlet源文件和字节码文件放到这个目录下
* **运行Tomcat**  
1）配置java_home或jre_home(java运行环境)  
2）双击bin目录下的startup.bat文件(可在dos下进入目录下运行)  
3）浏览器打开localhost:8080
4）启动会占用8080端口，所以一个Tomcat应用只能启动一次，否则会抛出端口被占用的异常
* **关闭Tomcat**  
运行shutdown.bat(可在dos下进入目录下运行)
* **修改Tomcat端口号**  
修改conf下的server.xml中的<Connector port>配置信息，修改Tomcat服务器的端口号  

        `<Connector port="8080" protocol="HTTP/1.1"
                   connectionTimeout="20000"
               	       redirectPort="8443" />`
* **在任意目录启动tomcat**  
1）将bin目录的地址存在Path环境变量中  
2）环境变量设置catalina_home  
3）启动tomcat服务器的是catalina.bat文件  
catalina start  
catalina run  
catalina stop
***
# 见过的另一种版本的Tomcat
* **下载安装包，直接安装**
* **打开bin目录下的Tomcat7w.exe可执行文件**
* **可以直接启动Tomcat**
***