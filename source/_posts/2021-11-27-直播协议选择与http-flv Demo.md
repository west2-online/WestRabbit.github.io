---
title: 直播协议选择与http-flv Demo
date: 2021-11-25
tags:
     - 19级
     - flv
     - 直播
author: 俞润鼎
---


# 直播协议选择与http-flv Demo
# 一、概述
面对网络带宽提升，直播成为不少网站、软件的必备功能。而流传输协议的选择将对直播效果产生巨大影响。
这里我们将对主流的几个协议进行简单的分析，并提供一个基础的应用实现。



# 二、推流方式
RMTP（Real Time Message Protocol）是目前主流的推流协议，他具有较低的时延，和高稳定性。
同时，有较多的相关实现，便于使用。具体内容将在后续分析。




# 三、拉流方式
+ HLS(Http Living Streaming)<br/>
HLS是基于HTTP协议的，实现方式是，在服务器收到视频流的时候，根据一个分段时间将视频缓存，并封包成一个ts文件，
同时维护一个m3u8文件用于保存视频的分段信息。然后对于客户端而言，就是在根据m3u8文件
来播放一个个视频片段。而对于每次视频片段的请求，都需要因此TCP握手，这就使得分段的时间大小不仅影响了时延，
同时频繁的请求还会额外占用资源。<br/>
官方的建议分段是10s。而直播的时延必然不止分段大小，通常在20s以上。<br/>
但其分段的优势使得视频源(如清晰度)的切换相对平滑。而且也得到了较好的浏览器等的支持。

+ RTMP(Real Time Messaging Protocol)<br/>
RTMP实际上是使用TCP作为传输协议，所以具有高可靠性，但与HLS不同的是，他使用的是长链接，因此只需要一次握手
即可重复传输数据。它将数据分为一个个的块，然后一个个发送。<br/>
由于是长连接，因此对服务器的资源消耗是比较大的，因此常是作为推流方式。<br/>
但其时延显著低于HLS协议，甚至能够达到1-3s。同时，由于不是http协议，可以防止http直接下载，具有较好的保密性
(技术门槛提高)。

+ RTSP(Real-Time Stream Protocol)<br/>
RTSP类似于RTMP，但是更类似于FTP。它通过RTP传输数据，通过RTSP本身发送控制信息，而不像RTMP，直接将控制命令与数据一起封包。<br/>
如此方式使得RTSP在实现直播的时候比较复杂，并且与RTMP一样，都具有累积时延<br/>
但也具有实时性好，稳定性高的特点。

+ HTTP<br/>
最后是http协议的方式。我们通过将直播的流数据(通过RTMP推流)直接打包成flv文件，然后模拟HTTP的方式发送。
这种方式类似于HLS，但是其通过http的方式传输flv文件，因此在第一次握手后，客户端即可不断下载服务器的flv文件(直播流)，服务器负担小。<br/>
但是与HLS相似，都具有保密性弱的劣势。<br/>
不过，它的时延可以达到与RTMP相似的程度。另外，原本需要使用Flash算是一个劣势，
但是，小破站NB，感想b站程序员开源的flv.js使得html5播放flv文件不再依赖flash。



# 四、直播Demo实现
+ 这里通过obs+nginx+nginx-rtmp-module实现rtmp推流+http_flv拉流。
+ 同时，依赖nginx-rtmp-module还实现了推流身份验证。
+ 具体安装过程自行百度。

配置文件如下
Nginx配置1（rtmp推流配置）
```shell script
#user  nobody;
worker_processes  1;
events {
    worker_connections  1024;
}
rtmp_auto_push on;
rtmp_auto_push_reconnect 1s;
rtmp_socket_dir /tmp;

rtmp {
    out_queue           4096;
    out_cork            8;
    max_streams         128;
    timeout             15s;
    drop_idle_publisher 15s;
    log_interval 5s; #interval used by log module to log in access.log, it is very useful for debug
    log_size     1m; #buffer size used by log module to log in access.log
    server {
        listen 1935;
        server_name localhost; #for suffix wildcard matching of virtual host name
        application http_flv {
            live on;
            on_publish http://xxx.com/auth;  # 推拉开始身份验证地址
            on_publish_done http://xxx.com/on_publish_done;  # 推拉结束回调地址
            gop_cache on; #open GOP cache for reducing the wating time for the first picture of video
        }
    }
}
```
Nginx配置2（http拉流配置）
```shell script
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        location /live {
            flv_live on; #open flv live streaming (subscribe)
            chunked_transfer_encoding  on; #open 'Transfer-Encoding: chunked' response
            add_header 'Access-Control-Allow-Origin' '*'; #add additional HTTP header
            add_header 'Access-Control-Allow-Credentials' 'true'; #add additional HTTP header
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

obs推流地址配置
![obs推流](./p1.png)

后端处理
由于nginx是通过post的方式请求的，所以通过该方法获取推流的相关信息。其中"推流地址"对应"swfurl"，"秘钥"对应"name"<br/>
这里我们将在播的主播存于Redis中，以加速查询在播用户。

```python
@api.route('/auth', methods=['POST'])
def auth():
    # 打开数据库连接
    data = request.form
    url = data.get('swfurl')    # 推流请求URL
    pull_key = data.get('name') # 推流秘钥
    if url and pull_key:
        lst_query = dict(urllib.parse.parse_qsl(urllib.parse.urlparse(url).query))
        push_key = lst_query.get('key')

        if push_key is not None:
            # 验证秘钥
            room = Room.query.filter(Room.push_key == push_key, Room.pull_key == pull_key).first()
            types = RoomType.query.filter(RoomType.id == room.type_id).first()
            
            # 分类存放在播主播
            if room and types:
                rd.sadd("liver", set_room_key(room))
                rd.sadd(types.mname, set_room_key(room))
                return Response(response='success', status=200)  # 返回200状态码

    return Response(status=500)  # 返回500状态码
```

最后前端通过js获取拉流地址
```javascript


function LiveUrl() {
    var params = {};
    var roomname = GetRoomName(location.href);
    var roomurl = null;
    $.ajax({
        url: RequestUrl+roomname,   // 请求地址
        type: "post",
        data: params,
        async:false,
        dataType: "json",
    }).done(function (ret) {
        if (!ret['code']) {
            var room_data = ret['data'];
            roomurl = liveUrl+"&stream="+room_data['url'];
            RoomSet(room_data);

        } else {
            alert(ret['msg']);
        }
    });
    return roomurl
}

function IsLiveRound() {

    if(!IsLive(GetRoomName(location.href))){
        console.log(IsLive(GetRoomName(location.href)));
        $("#unlive").html("主播暂时不在哦");
    }
    else{
        $("#unlive").html("");
    }
}
```

# 五、效果展示
![效果1](./p2.png)





