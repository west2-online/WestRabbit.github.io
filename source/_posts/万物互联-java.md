---
layout:      post
title: 万物互联---Java与Esp8266的梦幻联动
subtitle:   "Java后台"
date:       2020-11-26
author: "Oscar"
catalog:     true
tags:
     - 19级
     - Java
     - Web
---
软硬结合是个漫长的道路，传透模式是男人的浪漫（之一）。11月24号，有个朋友问我发生什么事情了，我说怎么回事，给我发了几张截图，我一看！嗷，原来是昨天，我拿到了刚到的Esp8266-01和烧录模块，一个3cm，一个4cm......&nbsp;&nbsp;——前言

## 实现

### Idea 的产生

在一所大专里看见外包团队在保安亭里安装**停车记录器**（大概是识别进出车辆的车牌号，进行计费和分配车位），稍微了解了一下，这个团队除了物理器械都是自己设计的。我一听啪的就站起来了，很快啊，*idea*就有了，或许是突然悟了，虽然说做软件是基本操作，但实现软硬互动也未尝不是代码的另一种风格

以下是三端的基本功能
java : 提供基本的交互后台
Esp8266 : 与后台交互
android : 端控制后台数据更新

**这一篇先介绍java模块**
### 1.搭建javaweb，完成简单的websocket
毕竟是介绍代码的，不能一上来就介绍*Esp8266*电路原理
普通的javaweb搭建我就不细说了，这样的一个小项目用不到高级的框架的。
简简单单，直接上码

``` 
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.OutputStream;
import java.net.Socket;
//author：Oscar
public class SendServlet extends HttpServlet {
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        //这里要放主要操作
    }
}
```
``` 
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

//author：Oscar
public class SocketServer {
    /**端口号*/
    private  static  final int  port = 80;
    /** 线程池*/
    private ExecutorService mExecutorService;
    /**ServerSocket对象*/
    private ServerSocket mServerSocket;
    /**存储socket*/
    public static Map<String, Socket> socketMap =new HashMap<>();
    private String ip;

    public SocketServer() {
        try {
            System.out.println("socket启动");
            //设置socket端口
            mServerSocket = new ServerSocket(port);
            //创建线程池
            mExecutorService = Executors.newCachedThreadPool();
            // 用来临时保存客户端连接的Socket对象
            Socket client = null;
            while(true){
                client = mServerSocket.accept();
                ip = client.getInetAddress().getHostAddress();
                System.out.println("ip="+ip);
                socketMap.put("1",client);
                System.out.println(client+"------");
                mExecutorService.execute(convertData);
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public Runnable convertData = new Runnable(){

        @Override
        public void run() {
            System.out.println("convertData");
            int result;
            try {
                Socket client = socketMap.get("1");
                while (true) {
                    InputStream inputStream = client.getInputStream();
                    OutputStream outputStream =client.getOutputStream();
                    while(inputStream.available()>0) {
                        byte[] data = new byte[inputStream.available()];
                        inputStream.read(data);
                        String resultData = new String(data);
                        resultData = replaceBlank(resultData);
                        System.out.println("resultData=" + resultData);
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    };
    
}
```
代码到这会就有人质疑了，这不死循环嘛，我说我这个有用，这是~~化劲儿~~方便透传，有念头就写出来的就是要讲究~~四两拨千斤~~方便

当然，这么做有它的弊端，比如`color{red}{mServerSocket = new ServerSocket(port);`
这一行，导致了部署端口和应用端口不同，一个项目要用到两个端口，一个端口还没用，加上两个监听端口，，，挺费端口的

另一个很大的弊端在实际操作后体现的更加明显，正常的websocket会对进入的用户进行临时的管理，而这个就比较草率了，如果不停的连接会出现内存无法释放服务器受不了的情况

当然，就这些弊端比起方便来说 **不值一提**

顺便贴一个有释放有管理的代码
``` 
//auther:Oscar
 private static final Logger log = LoggerFactory.getLogger(WebSocketServer.class);

    //静态变量，用来记录当前在线连接数。应该把它设计成线程安全的。
    public static int onlineCount = 0;
    //concurrent包的线程安全Set，用来存放每个客户端对应的MyWebSocket对象。
    public static CopyOnWriteArraySet<WebSocketServer> webSocketSet = new CopyOnWriteArraySet<WebSocketServer>();

    //与某个客户端的连接会话，需要通过它来给客户端发送数据
    public Session session;

    //接收参数中的用户ID
    public Long userId;

    //接收用户中的平台类型
    public Integer platformType;


    /**
     * 连接建立成功调用的方法
     * 接收url中的参数
     */
    @OnOpen
    public void onOpen(Session session, @PathParam("platformType") Integer platformType, @PathParam("userId") Long userId) {
        this.session = session;
        this.userId = userId;
        this.platformType = platformType;
        webSocketSet.add(this);     //加入set中
        addOnlineCount();           //在线数加1

        log.info("有新连接加入！当前在线人数为" + getOnlineCount() + "  userId==== " + userId + "  platformType==== " + platformType);
        try {
            sendMessage("连接成功");
        } catch (IOException e) {
            log.error("websocket IO异常");
        }
    }

    /**
     * 连接关闭调用的方法
     */
    @OnClose
    public void onClose() {
        webSocketSet.remove(this);  //从set中删除
        subOnlineCount();           //在线数减1
        log.info("有一连接关闭！当前在线人数为" + getOnlineCount());
    }

    /**
     * 收到客户端消息后调用的方法
     *
     * @param message 客户端发送过来的消息
     */
    @OnMessage
    public void onMessage(String message, Session session) {
        log.info("来自客户端的消息:" + message);
    }

    /**
     * @param session
     * @param error
     */
    @OnError
    public void onError(Session session, Throwable error) {
        log.error("发生错误" + error);
        error.printStackTrace();
    }


    public void sendMessage(String message) throws IOException {
        this.session.getBasicRemote().sendText(message);
    }


    /**
     * 私发
     *
     * @param message
     * @throws IOException
     */
    public static void sendInfo(Long userId, String message) throws IOException {
        for (WebSocketServer item : webSocketSet) {
            try {
                if (item.userId.equals(userId)) {
                    item.sendMessage(message);
                }
            } catch (IOException e) {
                continue;
            }
        }
    }


    /**
     * 群发自定义消息
     */
    public static void sendInfos(String message) throws IOException {
        log.info(message);
        for (WebSocketServer item : webSocketSet) {
            try {
                item.sendMessage(message);
            } catch (IOException e) {
                continue;
            }
        }
    }

    public static synchronized int getOnlineCount() {
        return onlineCount;
    }

    public static synchronized void addOnlineCount() {
        WebSocketServer.onlineCount++;
    }

    public static synchronized void subOnlineCount() {
        WebSocketServer.onlineCount--;
    }

```
有代码洁癖的可以参考以上代码
### 初始化
我们的websocket整到post里去了，还创建了队列，这必然要有Listener的
``` 
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
//author：Oscar
public class SocketListener implements ServletContextListener {

    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                new SocketServer();
            }
        }).start();
    }

    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {

    }
}
```



### 关于对Esp8266传来数据的处理
回车，占位都要换掉的
对于**硬件发送的数据**，特别是*Esp8266*，一定要小心谨慎，如果觉得不好判断的话可以先建立连接，打印出传过来的数据，再考虑如何**转义**
当然，这就涉及到http的知识点了，一般我们都是前后端http，https，对于硬件的数据传输，没有*\n\t\r*一般都不符合规范的，并且对字符的长度，参数的长度都有**严格**的要求。
``` 
//author：Oscar
public static String replaceBlank(String str) {
        String dest = "";
        if (str!=null) {
            Pattern p = Pattern.compile("\\s*|\t|\r|\n");
            Matcher m = p.matcher(str);
            dest = m.replaceAll("");
        }
        return dest;
    }
```
当时数据处理失败，跑去问了软工的dalao，她说她不会硬件，又跑去问物信的dalao，她说她不会软件。。。。
岂可修，只好自己琢磨了

### web.xml的配置

``` 
<servlet>
        <servlet-name>send</servlet-name>
        <servlet-class>SendServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>send</servlet-name>
        <url-pattern>/send.jsp</url-pattern>
    </servlet-mapping>
    <listener>
        <listener-class>SocketListener</listener-class>
    </listener>
```

