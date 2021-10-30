---  
title: Java的BIO与NIO模型  
date: 2021-10-29  
tags: 
   - Java  
   - D届  
author: Gatsby
---
# Java的I/O模型
**来聊一聊Java的BIO和NIO及一些底层原理，有时间下期更新Netty相关内容~**
## 模型基本说明

- 什么是I/O模型：就是用什么样的通道进行数据发送和接收，很大程度上决定了程序通信的性能
- Java共支持3种网络编程模型I/O模式：**BIO、NIO、AIO**
## 传统Java的I/O模型
**BIO：同步并阻塞（传统阻塞型）**

- 服务器为每一个新连接建立一个**新线程**进行处理，如果连接没有数据也会**阻塞等待**（不做任何事情）造成不必要的线程开销（示意图如下）   PS：BIO -> blocking io

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1634968592403-19ea959e-9012-45f0-b473-6add16e56504.png#clientId=u8dd99deb-8deb-4&from=paste&height=540&id=u2058aa70&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1080&originWidth=1692&originalType=binary&ratio=1&size=156893&status=done&style=none&taskId=u6c5af940-efef-4856-b1f0-bf1f5517e8a&width=846)
## 现在Java的I/O模型
**NIO：同步非阻塞**

- 服务器的一个线程能处理多个请求（连接），客户端发送的连接请求会注册到多路复用器上，多路复用器轮询到连接有I/O请求就进行处理（示意图如下） PS：NIO -> non-blocking io

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1634969539437-85b2dbeb-d501-47a8-9228-9d1850a79dff.png#clientId=u8dd99deb-8deb-4&from=paste&height=464&id=u858182a6&margin=%5Bobject%20Object%5D&name=image.png&originHeight=928&originWidth=1728&originalType=binary&ratio=1&size=164506&status=done&style=none&taskId=u2d984264-cd7c-462d-ab8a-6da46e4139e&width=864)
**AIO：异步非阻塞（非重点）**

- 引入了异步通道的概念，采用Proactor模式，简化了程序编写，有效的请求才会启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时长较长的应用
## BIO、NIO、AIO使用场景分析

- BIO适用于连接数小且固定的架构，对服务器资源占用高，一般只在JDK1.4以前（唯一选择）
- NIO适用于连接数目多且连接比较短的架构，如聊天服务器，弹幕系统，编程复杂，JDK1.4开始才有
- AIO适用于连接数目多且连接比较长的架构，如相册服务器，充分调用OS参与并发操作，JDK1.7后出现
# Java BIO 编程
## BIO基本介绍

- Java BIO就是传统的Java I/O编程，其相关类和接口在 java.io
- BIO（Blocking IO）：同步阻塞，服务器为每个连接请求建立新的线程，如果没有事情做也会阻塞等待，直到断开链接，这样会造成**不必要的线程开销**
- 对于BIO的架构可以通过线程池机制改善（多个客户连接服务器）
- BIO**占用资源高**，适用于连接数比较小且固定的架构，但是代码简单
## Java BIO 工作机制
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1634970912928-f704d2a7-bb6d-4237-9a21-098c95363e80.png#clientId=u8dd99deb-8deb-4&from=paste&height=469&id=u108b2819&margin=%5Bobject%20Object%5D&name=image.png&originHeight=937&originWidth=1487&originalType=binary&ratio=1&size=92727&status=done&style=none&taskId=u1323aeeb-fa47-4aca-b8c4-3ff23b56910&width=743.5)
**BIO工作流程**

1. 服务端启动一个**ServerSocket**
1. 客户端启动**Socket**对服务器进行通信，服务器对每一个客户端线程之间建立通讯
1. 客户端线程请求结束后，断开链接，服务端线程结束
## Java BIO问题分析

- 每个请求都需要创建独立的线程，为客户端数据Read/Write
- 当并发数大的时候，要创建大量线程来处理链接，系统资源占用大
- 建立连接后，线程如果暂时没有数据可读，就阻塞在Read上，造成线程资源浪费
# Java NIO 编程
## NIO基本介绍

- Java NIO 全称 **Java non-blocking IO**是JDK1.4开始提供的新API，是**同步非阻塞**的
- NIO相关类都放在java.nio包下，并对原**java.io**包中的很多类进行改写
- NIO有三大核心部分：**Channel（通道）、Buffer（缓冲区）、Selector（选择器）**
- NIO是**面向缓冲区，或者面向块编程**的，不是简单的read和write，数据读取到一个它稍后处理的缓冲区，也可按需要在缓冲区前后移动读取数据
- NIO的**非阻塞模式**：一个线程从某通道读取数据的时候，**仅能得到目前可用的数据**，当没有可用的数据时，什么都不会获取，会去做别的事情（**因为数据是块的，所以等它攒到一定大小直到可用的时候再给就行，而不会因为数据是散装的要一直阻塞等待浪费线程！！！**）
- 这样就能将很多请求交由一个线程处理，大幅度减少线程开销
## NIO和BIO

- BIO以**流**的形式处理数据，NIO以**块**的形式处理数据
- BIO是**阻塞的**，NIO是**非阻塞的**
- BIO的流是**字节流**和**字符流**，**而NIO的块基于**Channel（通道）**和**Buffer（缓冲区）**读取数据**，**Selector（选择器）**监听多个通道的**事件（连接请求，数据到达）**
## NIO三大核心
### 关系图
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1635350190420-d6f28f65-c638-4f7a-b43b-cce036ec56fe.png#clientId=u39e72bc5-ad24-4&from=paste&height=388&id=u1e9a116c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=775&originWidth=713&originalType=binary&ratio=1&size=64457&status=done&style=none&taskId=uda6c6720-f25a-4f3a-b442-61e2c8ec45f&width=356.5)

- 每一个Channel都会对应一个Buffer
- Selector对应一个服务器线程，一个线程对应多个Channel
- Selector选择哪个是由Channel的**事件Event**决定的
- Selector会根据事件安排在各个通道上**切换**
- Buffer是缓冲区，其实就是一个内存块，底层就是一个数组（bytebuffer）
- Buffer的读写是双向，既可以读也可以写，像底层操作系统的系统通道也是双向的
### 缓冲区（Buffer）
**Buffer的本质 -> 可以读写的数据内存块（容器对象）**
**数据的读写必须经过buffer**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1635350731165-5f50e014-141c-469f-a466-65ff54c64e5a.png#clientId=u39e72bc5-ad24-4&from=paste&height=187&id=u6cfaeedb&margin=%5Bobject%20Object%5D&name=image.png&originHeight=374&originWidth=1490&originalType=binary&ratio=1&size=44597&status=done&style=none&taskId=ub208bdfc-9bdb-45af-8ad9-b8b347dabd0&width=745)
Buffer类的基本信息
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1635350922717-25b3639b-745b-413a-9b85-2c9413106b76.png#clientId=u39e72bc5-ad24-4&from=paste&height=278&id=u3832962c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=556&originWidth=1733&originalType=binary&ratio=1&size=325206&status=done&style=none&taskId=u80d89859-a964-4d80-9b17-764ad74a601&width=866.5)
Buffer类相关方法（比较灵活）
![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1635350965708-f70069ce-71b8-4df0-828b-a2ef4875948b.png#clientId=u39e72bc5-ad24-4&from=paste&height=442&id=ue7394780&margin=%5Bobject%20Object%5D&name=image.png&originHeight=884&originWidth=1709&originalType=binary&ratio=1&size=816120&status=done&style=none&taskId=u3ffdeddf-a714-4b4d-b769-351a618cc86&width=854.5)
### 通道（Channel）
**NIO通道与BIO的流的区别**

- **通道可以同时进行读写，流只能读或者写**
- **通道可以异步读写数据，流只能同步**
- **通道可以从缓冲区读&写数据**

**常用Channel类**

- BIO的Stream是**单向**的，NIO的Channel是**双向**的
- 常用的Channel类：**ServerSocketChannel**（类似ServerSocket）和**SocketChannel**（类似Socket）
- **Filechannel ->文件读写通道**
- **DatagramChannel -> UDP数据读写通道**
- **ServerSocketChannel & SocketChannel -> TCP读写通道**

### Selector（选择器）

- Java的NIO，用非阻塞IO模式，得益于**Selector**
- **多个Channel以事件的方式注册到同一个Selector -> 这样Selector可监控多个通道是否有事件发生 -> 一个线程管理多个通道的连接和请求**
- **只有有事件发生时才回去读写，减少线程开销（线程上下文切换）**
- 线程在非阻塞IO的空闲时间在其他有需要的通道上执行IO操作，节约了时间
- **Netty的IO线程NioEventLoop（事件）**

**示意图**

![image.png](https://cdn.nlark.com/yuque/0/2021/png/22873239/1635407374160-ebd36ce9-7ebd-45b9-ab42-01c4ab154caf.png#clientId=u108c2a2c-2a37-4&from=paste&height=359&id=uf2d3d246&margin=%5Bobject%20Object%5D&name=image.png&originHeight=717&originWidth=737&originalType=binary&ratio=1&size=239011&status=done&style=none&taskId=ua38dee8e-9788-4f38-a7ce-0b3b57e2b16&width=368.5)
# 参考资料

- 尚硅谷Netty【韩顺平】：[https://www.bilibili.com/video/BV1DJ411m7NR](https://www.bilibili.com/video/BV1DJ411m7NR)
- 推荐一个I/O形象解释：[https://blog.csdn.net/szxiaohe/article/details/81542605](https://blog.csdn.net/szxiaohe/article/details/81542605)
