---
title: Golang的Redis简单使用及集群配置
date: 2021-11-24
tags: 
    - 19级
    - Golang
author: 北朔
---

#Golang的Redis简单使用及集群配置

###1.Go-Redis库的安装以及Redis的安装

#### Go-Redis库的安装
Github地址：https://github.com/go-redis/redis

可以自己go get 或者 git 亦或直接下载放到对应文件夹里即可。

#### Redis的安装
Github地址：https://github.com/tporadowski/redis/releases

找最新的zip直接下载即可。

###2.简单使用
在代码跑之前需要打开服务端，也就是刚才下载的redis文件夹里面找到redis-server.exe，打开即可。

#### redis客户端：
```
client := redis.NewClient(&redis.Options{
		Addr: "127.0.0.1:6379",//Redis的端口地址
		Password: "",//不用密码
		DB: 0,//采用默认DB
	})
var ctx = context.Background()
```
这里的Addr是默认的6379，如果要改的话~~(百度吧)~~，这里的ctx是各种操作需要的第一个参数。

只要Addr一样，就可以直接访问之前存放的数据，与client在哪个go文件使用无关，因此可以存放一些即时数据不需要用到MySQL。

Redis的存放是按照哈希存的，和Map数据结构类似。

#### Redis-Set
```
	ping, err := client.Ping(ctx).Result()//检测连接
	fmt.Println(ping, err)
    //第二个参数为Key，第三个参数为Value，第四个参数为过期时间，0为不过期
	err = client.Set(ctx,"key","myKey",0).Err()
	if err != nil {
		panic(err)
	}

```
#### Redis-Get
```
    //第二个参数为Key
    val, err := client.Get(ctx,"key").Result()
    if err != nil {
        panic(err)
    }
    fmt.Println("key", val)
```
#### Redis-Del
```
    //第二个参数为Key
    client.Del(ctx,"key")
```

#### 一些容器的使用
List：
```
    //List容器
    client.RPush(ctx,"list","val1")
	client.RPush(ctx,"list","val2")
	client.RPush(ctx,"list","val3")
    //取List长度
	rLen, err := client.LLen(ctx,"list").Result()
    //遍历
	lists,_ := client.LRange(ctx,"list",0, rLen - 1).Result()
	fmt.Println("list",lists)
    //修改下标为0位置的值
	client.LSet(ctx,"list",2,"change")

	lists,_ = client.LRange(ctx,"list",0, rLen - 1).Result()
	fmt.Println("list",lists)
    //清空List
	for  ; rLen > 0 ; rLen-- {
		client.LPop(ctx, "list")
	}
```
其他容器大致和List的操作是一样的，比如Map，Set等等。

它们的对应函数不一样罢了，对应参数顺序大致一样。

#### Demo
```
func main(){
	client := redis.NewClient(&redis.Options{
		Addr: "127.0.0.1:7000",
		Password: "",
		DB: 0,
	})
	var ctx = context.Background()
	ping, err := client.Ping(ctx).Result()//检测连接
	fmt.Println(ping, err)
	//第二个参数为Key，第三个参数为Value，第四个参数为过期时间，0为不过期
	err = client.Set(ctx,"key","myKey",0).Err()
	if err != nil {
		panic(err)
	}

	//第二个参数为Key
	val, err := client.Get(ctx,"key").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("key", val)

	val2, err := client.Get(ctx,"key1").Result()
	if err == redis.Nil {
		fmt.Println("key1 does not exists")
	} else if err != nil {
		panic(err)
	} else {
		fmt.Println("key1", val2)
	}
	
	//第二个参数为Key
	client.Del(ctx,"key")

	val3, err := client.Get(ctx,"key").Result()
	if err == redis.Nil {
		fmt.Println("key does not exists")
	} else if err != nil {
		panic(err)
	} else {
		fmt.Println("key", val3)
	}
    //List容器
    //第二个参数为Key，第三个参数为Value值，可以输入多个value值一次性Push
    //client.RPush(ctx,"list","val1","val2","val3")
    client.RPush(ctx,"list","val1")
	client.RPush(ctx,"list","val2")
	client.RPush(ctx,"list","val3")
    //取List长度
	rLen, err := client.LLen(ctx,"list").Result()
    //遍历
	lists,_ := client.LRange(ctx,"list",0, rLen - 1).Result()
	fmt.Println("list",lists)
    //修改下标为0位置的值
	client.LSet(ctx,"list",2,"change")

	lists,_ = client.LRange(ctx,"list",0, rLen - 1).Result()
	fmt.Println("list",lists)
    //清空List
	for  ; rLen > 0 ; rLen-- {
		client.LPop(ctx, "list")
	}
	rLen, err = client.LLen(ctx,"list").Result()
	lists,_ = client.LRange(ctx,"list",0, rLen - 1).Result()
	fmt.Println("list",lists)

	client.RPush(ctx,"list","val1","val2","val3")
	rLen, err = client.LLen(ctx,"list").Result()
	lists,_ = client.LRange(ctx,"list",0, rLen - 1).Result()
	fmt.Println("list",lists)

	client.Close()
}
```
输出结果：
```
PONG <nil>
key myKey
key1 does not exists
key does not exists
list [val1 val2 val3]
list [val1 val2 change]
list []
list [val1 val2 val3]
```
### 3.Redis集群配置

因为Redis集群是半数淘汰，因此至少需要3个节点才可以使用。

这里采用三主三从，可以更好的避免节点失误导致集群不可用的情况发生。

#### 6个节点配置文件需要修改的地方
```
port 你要设置的端口号                     #(例如7000)           
appendonly yes                          #数据的保存为aof格式
appendfilename "appendonly.7000.aof"    #数据保存文件   
cluster-enabled yes                     #是否开启集群                                    
cluster-config-file nodes.7000.conf     #集群配置文件
cluster-node-timeout 15000              #节点超时时间，如果一个master节点不可到达超过了指定时间，则认为它失败了。
cluster-slave-validity-factor 10        #如果设置为0，则一个slave将总是尝试故障转移一个master。如果设置为一个正数，那么最大失去连接的时间是node timeout乘以这个factor。
cluster-migration-barrier 1             #一个master和slave保持连接的最小数量（即：最少与多少个slave保持连接）
cluster-require-full-coverage yes       #如果设置为yes，这也是默认值，如果key space没有达到百分之多少时停止接受写请求。如果设置为no，将仍然接受查询请求，即使它只是请求部分key。 
```

#### Windows启动集群服务
将上述配置文件保存到Redis目录下，并使用这些配置文件安装服务并启动
```
安装：
Redis路径\redis-server --service-install Redis路径\conf文件名.conf --service-name redis文件端口号
例如：
D:\Redis\redis-server --service-install D:\Redis\redis.7000.conf --service-name redis7000
启动：
Redis路径\redis-server --service-start --service-name Redis端口号
例如：
D:\Redis\redis-server --service-start --service-name Redis7000
```
启动后图片：

![avatar](./server-show.png)

下载ruby：

url：https://rubyinstaller.org/downloads/

选择without devkit的最新版本

下载redis驱动：

url:https://rubygems.org/gems/redis/versions

选择兼容版本即可，我用的是3.2.0

安装驱动：
```
gem install --local gem文件路径\filename.gem  
```

而现在的cluster安装方式不推荐使用redis-trib.rb，而是使用redis-cli

下载正确的redis-trib.rb:
url:https://pan.baidu.com/s/1hpOu7fGD9pCzpXQ6fFXJzg 提取码：v00z

开始cluster配置：

打开命令行，移动到redis-trib.rb目录之后，输入命令：

```
redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
```

会弹出：
```
Can I set the above configuration? (type 'yes' to accept):
```
输入yes回车即可

Mine：
```
C:\Users\WIN10\Desktop\Redis>redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
M: d0e194b5a53a79782c49ef44e3643ffce3842a67 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 543e52694edf6db069b912fe0f1252f0275189f6 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: 595906624067ffff9f204a5ccd2fb1c1911344a7 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
S: 633c5b3ae785086c6c6041724c0b81be92e00ed0 127.0.0.1:7003
   replicates d0e194b5a53a79782c49ef44e3643ffce3842a67
S: fe62faa1ecb26ca968433feab937c5833baf1db0 127.0.0.1:7004
   replicates 543e52694edf6db069b912fe0f1252f0275189f6
S: 20d97ec5b8a129366f7a7496a85404efb8e71371 127.0.0.1:7005
   replicates 595906624067ffff9f204a5ccd2fb1c1911344a7
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join..
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: d0e194b5a53a79782c49ef44e3643ffce3842a67 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 543e52694edf6db069b912fe0f1252f0275189f6 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: 595906624067ffff9f204a5ccd2fb1c1911344a7 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
M: 633c5b3ae785086c6c6041724c0b81be92e00ed0 127.0.0.1:7003
   slots: (0 slots) master
   replicates d0e194b5a53a79782c49ef44e3643ffce3842a67
M: fe62faa1ecb26ca968433feab937c5833baf1db0 127.0.0.1:7004
   slots: (0 slots) master
   replicates 543e52694edf6db069b912fe0f1252f0275189f6
M: 20d97ec5b8a129366f7a7496a85404efb8e71371 127.0.0.1:7005
   slots: (0 slots) master
   replicates 595906624067ffff9f204a5ccd2fb1c1911344a7
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
验证连接：
```
输入：
redis-cli -p 7000 cluster nodes
响应：
20d97ec5b8a129366f7a7496a85404efb8e71371 127.0.0.1:7005@17005 slave 595906624067ffff9f204a5ccd2fb1c1911344a7 0 1637803028599 6 connected
633c5b3ae785086c6c6041724c0b81be92e00ed0 127.0.0.1:7003@17003 slave d0e194b5a53a79782c49ef44e3643ffce3842a67 0 1637803027508 4 connected
595906624067ffff9f204a5ccd2fb1c1911344a7 127.0.0.1:7002@17002 master - 0 1637803028000 3 connected 10923-16383
d0e194b5a53a79782c49ef44e3643ffce3842a67 127.0.0.1:7000@17000 myself,master - 0 1637803028000 1 connected 0-5460
543e52694edf6db069b912fe0f1252f0275189f6 127.0.0.1:7001@17001 master - 0 1637803026413 2 connected 5461-10922
fe62faa1ecb26ca968433feab937c5833baf1db0 127.0.0.1:7004@17004 slave 543e52694edf6db069b912fe0f1252f0275189f6 0 1637803029690 5 connected
或者输入：
redis-trib.rb check 127.0.0.1:7000
响应：
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: d0e194b5a53a79782c49ef44e3643ffce3842a67 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
S: 20d97ec5b8a129366f7a7496a85404efb8e71371 127.0.0.1:7005@17005
   slots: (0 slots) slave
   replicates 595906624067ffff9f204a5ccd2fb1c1911344a7
S: 633c5b3ae785086c6c6041724c0b81be92e00ed0 127.0.0.1:7003@17003
   slots: (0 slots) slave
   replicates d0e194b5a53a79782c49ef44e3643ffce3842a67
M: 595906624067ffff9f204a5ccd2fb1c1911344a7 127.0.0.1:7002@17002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
M: 543e52694edf6db069b912fe0f1252f0275189f6 127.0.0.1:7001@17001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: fe62faa1ecb26ca968433feab937c5833baf1db0 127.0.0.1:7004@17004
   slots: (0 slots) slave
   replicates 543e52694edf6db069b912fe0f1252f0275189f6
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

集群demo：
```
C:\Users\WIN10\Desktop\Redis>redis-cli -c -p 7000
127.0.0.1:7000> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:7002
OK
127.0.0.1:7002> set  hello world
-> Redirected to slot [866] located at 127.0.0.1:7000
OK
127.0.0.1:7000> set good nice
OK
127.0.0.1:7000> set apple iphone
-> Redirected to slot [7092] located at 127.0.0.1:7001
OK
127.0.0.1:7001> set banana orange
OK
127.0.0.1:7001> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
127.0.0.1:7002> get apple
-> Redirected to slot [7092] located at 127.0.0.1:7001
"iphone"
127.0.0.1:7001> get good
-> Redirected to slot [2195] located at 127.0.0.1:7000
"nice"
127.0.0.1:7000> set gay shit
OK
127.0.0.1:7000> get gay
"shit"
127.0.0.1:7000> flush all
(error) ERR unknown command `flush`, with args beginning with: `all`,
127.0.0.1:7000> flushall
OK
127.0.0.1:7000> get gay
(nil)
127.0.0.1:7000> get doo
(nil)
127.0.0.1:7000> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
127.0.0.1:7002> flush all
(error) ERR unknown command `flush`, with args beginning with: `all`,
127.0.0.1:7002> flushall
OK
127.0.0.1:7002> get apple
-> Redirected to slot [7092] located at 127.0.0.1:7001
"iphone"
127.0.0.1:7001> flushall
OK
127.0.0.1:7001> get gay
-> Redirected to slot [2398] located at 127.0.0.1:7000
(nil)
127.0.0.1:7000> get apple
-> Redirected to slot [7092] located at 127.0.0.1:7001
(nil)
127.0.0.1:7001> get banana
(nil)
127.0.0.1:7001> get good
-> Redirected to slot [2195] located at 127.0.0.1:7000
(nil)
127.0.0.1:7000> get key
-> Redirected to slot [12539] located at 127.0.0.1:7002
(nil)
```
#### demo解释

Redis集群中有16384个hash slots，为了计算给定的key应该在哪个hash slot上，我们简单地用这个key的CRC16值来对16384取模。（即：key的CRC16  %  16384）

Redis集群中的每个节点负责一部分hash slots，假设你的集群有3个节点，那么：

Node A contains hash slots from 0 to 5500

Node B contains hash slots from 5501 to 11000

Node C contains hash slots from 11001 to 16383

因此对不同的Key，存储的节点位置是不一样的，但会自动切换到对应的节点位置。
