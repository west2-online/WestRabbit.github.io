---
title: 跨域与同源策略探究
date: 2018-03-10 11:16:53
author: Chs97
tags:
	- 前端
	- 15级
---


### 背景

跨域这个问题前端开发者都接触过，网上的文章也非常多，但是昨天的腾讯二面给我留了非常深刻的印象，原来跨域能问出那么多花样，难怪所有面试官都喜欢和面试者来探讨这个问题。

### 跨域

#### 一、什么是跨域

简单的来说就是浏览器限制了向不同域发送ajax请求。

不同域体现在：域名、端口、协议不同

<!--more-->

#### 二、怎么解决跨域

##### 1.JSONP

JSONP在CORS出现之前，是最常见的一种跨域方案，在IE10以下的版本，是不兼容CORS的，所以如果有需要兼容IE10一下的，都会使用JSONP去解决跨域问题。

###### JSONP的基本原理：

动态向页面添加一个script标签，浏览器对脚本请求是没有域限制的，浏览器会请求脚本，然后解析脚本，执行脚本。通过这个我们就可以实现跨域请求。

```javascript
  function handleResponse(response) {
    console.log(response)
  }
  var script = document.createElement("script")
  script.src = "http://localhost:3000/?callback=handleResponse"
  document.body.insertBefore(script, document.body.firstChild)
```

研究一下这段代码，新建一个script标签，设置标签的src属性，动态插入标签到body后面。插入以后浏览器会请求src的内容，下载下来并执行。

那怎么通过回调handleResponse获得数据？src后面那段querystring又是干什么的？

如果请求得到的脚本里面的代码长这样

```js
handleResponse('hello world')
```

执行的时候是不是就可以通过回调得到data -> 'hello world' 

所以jsonp其实也需要后端的支持，这个queryString就是让后端知道你前端的回调方法，然后要返回怎样的脚本给前端。

``` javascript
const koa = require('koa')
const app = new koa()
app.use(async ctx => {
  ctx.body = `${ctx.query.callback}('hello world')`;
});
app.listen(3000)
```

这是node的一段代码，简单的体现出了jsonp后端的处理方式。

###### JSONP的缺点

只能发送get请求，感觉这不是正经的手段，而是一个奇淫巧技。

对于，为什么浏览器对于JavaScript不做同源策略这个问题，我觉得主要原因是需要后端的支持，他需要通过后端返回的内容，并且执行，产生回调才可以取得到结果。然而ajax不一样，js可以直接拿到ajax的返回结果。

还有一种可能就是，静态资源需要放到CDN上，或者引用一些公共的脚本，应该是需求导致。

##### 2.CORS

CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。

它允许浏览器向跨源服务器，发出[`XMLHttpRequest`](http://www.ruanyifeng.com/blog/2012/09/xmlhttprequest_level_2.html)请求，从而克服了AJAX只能[同源](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)使用的限制。

###### 什么是CORS？

CORS需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能，IE浏览器不能低于IE10。

整个CORS通信过程，都是浏览器自动完成，不需要用户参与。对于开发者来说，CORS通信与同源的AJAX通信没有差别，代码完全一样。浏览器一旦发现AJAX请求跨源，就会自动添加一些附加的头信息，有时还会多出一次附加的请求，但用户不会有感觉。

因此，实现CORS通信的关键是服务器。只要服务器实现了CORS接口，就可以跨源通信。

拿Koa举个栗子。在Koa中，我们只需要在Koa中加个中间件

```javascript
const koa = require('koa')
const cors = require('koa2-cors')
const app = new koa()
app.use(cors())
app.use(async ctx => {
  ctx.body = 'hello world';
});
app.listen(3000)
```

这样就能实现服务端CORS接口。

###### 两种请求

关于为什么有这两种请求的区别，我个人认为，浏览器发送预请求所消耗的资源会比简单请求还多，所以浏览器发送简单请求不需要发送预请求。而非简单请求，如果发送以后，然后被服务器拒绝了，所消耗的资源比预请求还多，所以在发送非简单请求之前，使用一个预请求来判断服务器是否允许跨域。

1.简单请求

同时满足一下条件为简单请求

（1) 请求方法是以下三种方法之一：

- HEAD
- GET
- POST

（2）HTTP的头信息不超出以下几种字段：

- Accept
- Accept-Language
- Content-Language
- Last-Event-ID
- Content-Type：只限于三个值`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`

如果content-type的值为`application/json`那么这个请求就是非简单请求，需要发送预请求。

2.非简单请求

不属于简单请求都是非简单请求，非简单请求就需要预请求。

###### 预请求

非简单请求会在正式请求之前发送一次预请求，这个请求是浏览器发的。浏览器先向服务器询问，当前的请求域是否在服务器的许可名单中，已经服务器允许哪一些方法，哪一些请求头。只有得到服务器的答复，并且之后发送的那个正式请求是被允许的（方法和请求头），浏览器才会发送这个正式请求。

"预检"请求用的请求方法是`OPTIONS`，表示这个请求是用来询问的。头信息里面，关键字段是`Origin`，表示请求来自哪个源。

除了`Origin`字段，"预检"请求的头信息包括两个特殊字段。

**Access-Control-Request-Method**

字面意思，内容是允许的请求方法

**Access-Control-Request-Headers**

同字面意思，内容是允许的额外的请求头

##### 3.通过iframe

没有特殊情况下，iframe的加载是没有跨域限制的。`<iframe>` 载入的任何资源是允许跨域的。我们可以通过几个手段，让iframe的内容，传递到父窗口中。

1.Window.name + iframe

- window.name属性值在文档刷新后依旧存在的能力（且最大允许2M左右）。
- 每个iframe都有包裹它的window。
- contenWindow返回的是`<iframe>`元素的window对象

```html
// localhost:3001/index.html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Document</title>
</head>
<body>
    <script>
    var iframe = document.createElement('iframe')
    iframe.style.display = 'none'
    var state = 0 // 设置状态防止页面无限刷新
    iframe.onload = function() {
      if (state == 1) {
        console.log(iframe.contentWindow.name)
        // 清除创建的iframe
        iframe.contentWindow.document.write('');
        iframe.contentWindow.close();
        document.body.removeChild(iframe);
      } else if (state == 0) {
        state = 1
        iframe.contentWindow.location = 'http://localhost:3001/proxy.html'; //加载完成，iframe指回当前域
        // 防止 Blocked a frame with origin "xxxx" from accessing a cross-origin frame.错误
      }
    }
    iframe.src = 'http://localhost:3000/';
    document.body.appendChild(iframe);
  </script>
</body>
</html>
```

```javascript
// localhost:3000,server
const koa = require('koa')
// const cors = require('koa2-cors')
const app = new koa()
// app.use(cors())
app.use(async ctx => {
  data = '\'hello world\''
  ctx.body = `
    <h1>${data}</h1>
    <script>
      window.name = ${data}
    </script>
  `;
});
app.listen(3000)
```

结果： 浏览器console 输出 hello world，表示我们在localhost:3001中拿到了localhost:3000的数据。



#### 三、为什么浏览器需要同源策略

考虑一下这个场景：你打开了A网站，并且登录了A网站，A网站也记录了你的cookie信息，然后你打开一个B网站，如果没有同源策略，B网站是可以直接请求A网站的接口的，有一些比如个人信息，他就可以通过get等方法，获得到你的信息，甚至可以post等操作去修改你的信息，这样你的账户安全是受到很严重的威胁的。所以浏览器需要同源策略来保证一定的安全，攻击手段例如CSRF和XSS，下面会详细讲。

#### 四、跨源网络访问

##### 1.通常允许跨域写操作

例如：links，重定向（传统页面的跳转），以及表单提交。特定少数的HTTP请求需要添加 [preflight](https://developer.mozilla.org/zh-CN/docs/HTTP/Access_control_CORS#Preflighted_requests)。

##### 2.通常允许跨域资源嵌入

- `<script src="..."></script>` 标签嵌入跨域脚本。语法错误信息只能在同源脚本中捕捉到。
- `<link rel="stylesheet" href="...">` 标签嵌入CSS。由于CSS的[松散的语法规则](http://scarybeastsecurity.blogspot.dk/2009/12/generic-cross-browser-cross-domain.html)，CSS的跨域需要一个设置正确的`Content-Type` 消息头。不同浏览器有不同的限制： [IE](http://msdn.microsoft.com/zh-CN/library/ie/gg622939%28v=vs.85%29.aspx), [Firefox](http://www.mozilla.org/security/announce/2010/mfsa2010-46.html), [Chrome](http://code.google.com/p/chromium/issues/detail?id=9877), [Safari](http://support.apple.com/kb/HT4070) (跳至CVE-2010-0051)部分 和 [Opera](http://www.opera.com/support/kb/view/943/)。
- `<img>`嵌入图片。支持的图片格式包括PNG,JPEG,GIF,BMP,SVG,...
- `video`和 `audio`嵌入多媒体资源。
- `@font-face` 引入的字体。一些浏览器允许跨域字体（ cross-origin fonts），一些需要同源字体（same-origin fonts）。
- `<iframe>` 载入的任何资源。站点可以使用[X-Frame-Options](https://developer.mozilla.org/zh-CN/docs/HTTP/X-Frame-Options)消息头来阻止这种形式的跨域交互。

曾经就遇到过字体文件跨域问题，在引入fontawesome的时候，webpack打包之后上传到服务器，浏览器加载不到字体文件，错误显示了跨域的问题。解决方案需要在nginx配置请求字体文件返回的请求头

```nginx
location ~* \.(eot|ttf|woff|svg|otf)$ {
     add_header Access-Control-Allow-Origin *;
}
```

##### 3.canvas中img的跨域

> HTML 规范中图片有一个 `crossorigin` 属性，结合合适的 [CORS](https://developer.mozilla.org/en-US/docs/Glossary/CORS) 响应头，就可以实现在画布中使用跨域<img> 元素的图像。

有一次有个需求，就是对页面进行截图，我的方案是html2canvas ,canvs2image，结果导致了部分图片截不下来，其原因就是canvas画布被污染了。

> 尽管不通过 CORS 就可以在画布中使用图片，但是这会**污染**画布。一旦画布被污染，你就无法读取其数据。例如，你不能再使用画布的 `toBlob()`, `toDataURL()` 或 `getImageData()` 方法，调用它们会抛出安全错误。

解决办法对画布中的图片配置`img.crossOrigin = "Anonymous";`

### CSRF与XSS

#### 一、XSS

> 跨站脚本（英语：Cross-site scripting，通常简称为：XSS）是一种网站应用程序的安全漏洞攻击，是代码注入的一种。它允许恶意用户将代码注入到网页上，其他用户在观看网页时就会受到影响。这类攻击通常包含了HTML以及用户端脚本语言。
>
> XSS其实是为了和CSS做区分吧。

在好几年前，XSS非常流行，可以用XSS获得用户的cookie，浏览器版本等信息，比如如果使用XSS获得到了管理员的cookie，那网站可就危险了。随着行业的发展，XSS越来越受重视，浏览器也对这种手段做了一定的预防，比如同源策略？CSP?

##### 1.XSS是什么

###### 反射型

反射型的攻击需要攻击者去欺骗用户点击，或者脚本当作url参数注入到页面中。

举个栗子

```html
<!DOCTYPE html>
<html>

<head>
  <meta charset="UTF-8">
  <title>Document</title>
</head>

<body>
  <script>
    var str = window.location.href.split('injection=')[1]
    document.write('<script>' +
      decodeURIComponent(str) +
      '<\/script>')
  </script>

</html>
```

浏览器输入url+?injection=alert(1)

这时候会弹出一个弹窗内容为1，这就是XSS注入的一种方式。

###### 存储型

存储型比反射型的危害更大，因为存储型是用户把攻击代码提交到数据库中，当别的用户访问时，数据库把攻击代码返回给用户，用户就会受到攻击。

[xss link&svg黑魔法](https://lorexxar.cn/2015/11/19/xss-link/)

##### 2.怎么预防XSS

1. 尽量不在特定地方输出不可信变量：script / comment / attribute / tag / style， 因为逃脱 HTMl 规则的字符串太多了。
2. 将不可信变量输出到 div / body / attribute / javascript tag / style 之前，对 `& < > " ' /` 进行转义
3. 将不可信变量输出 URL 参数之前，进行 URLEncode
4. 使用合适的 HTML 过滤库进行过滤。介绍个库[Secure XSS Filters](https://github.com/yahoo/xss-filters)
5. 预防 DOM-based XSS，见 [DOM based XSS Prevention Cheat Sheet](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
6. 开启 HTTPOnly cookie，让浏览器接触不到 cookie

#### 二、CSRF

##### 1.CSRF是什么

Cross-site request forgery 跨站请求伪造，也被称为 “one click attack” 或者 session riding，通常缩写为 CSRF 或者 XSRF，是一种对网站的恶意利用。CSRF 则通过伪装来自受信任用户的请求来利用受信任的网站。

##### 2.怎么预防

从它的原理来讲，我们的目的就是，不让已经登录A网站的用户，打开B网站后，受到攻击，也就是说，不让B网站向A网站发送有效的请求，获得用户在A网站的信息或者进行破坏。

###### Token

1.SSR

大部分SSR的网站都使用form表单进行提交内容的，这时候如果B网站也构造一个和A网站一样的form表单，一样可以提交内容到A网站。

我们只需要一个能验证身份的，能证明这个请求是从A发出来的，而不是B发出来的就可以了。如下面Codeforces的Login

```html
<form method="post" action="" id="enterForm"><input type="hidden" name="csrf_token" value="635faaad128c55d2849b89e80bf790a3">
            <table class="table-form">
                <input type="hidden" name="action" value="enter">
                <input type="hidden" name="ftaa" value="">
                <input type="hidden" name="bfaa" value="">
               
        <input type="hidden" name="_tta" value="243"></form>
```

在生成form的时候，后台插入了一个csrf_token用来验证请求来源，这个csrf_tokenB网站是拿不到的，所以这样就能验证请求是从A发出来的而不是B。

2.SPA

对于SPA，非SSR，无法在form中嵌入一个csrf_token，一般后台是不会把*Origin*设置成\*的，可以直接限制请求来源，有一点得注意的，请求必须保证不能用form构造出来，比如可以使用json交互。如果后台偷懒，像我一样，直接用个中间件没有设置啥的话，默认是不做跨域限制的。

这时候就将身份验证信息放在localStorage或者其他地方，总之不要放在cookie中，我们每次请求带上信息。B网站是读不到这些信息的，也就无法攻击了。

###### SameSite

关于这个SameSite好像使用得非常少。

SameSite-cookies是一种机制，用于定义cookie如何跨域发送。这是谷歌开发的一种安全机制，并且现在在最新版本（Chrome Dev 51.0.2704.4）中已经开始实行了。SameSite-cookies的目的是尝试阻止CSRF（Cross-site request forgery 跨站请求伪造）以及XSSI（Cross Site Script Inclusion (XSSI) 跨站脚本包含）攻击。

似乎，只有在chrome浏览器才有这个机制。

这个的原理是阻止第三方cookie的发送，比如：B网站向A网站的接口发送请求，是不会带上A网站已有的cookie的，而A网站向A网站的接口发送请求，还是会带上cookie的。这是浏览器的一种机制。具体的请看参考链接中的`再见，CSRF：讲解set-cookie中的SameSite属性`

###### 检查来源

验证referer字段

HTTP头有一个字段叫做referer它记录了该 HTTP 请求的来源地址。后台可以根据这个字段来判断这个请求是否是从网站A发出的，如果不是，就是不合法的请求。

##### 3.CSRF窃取

利用CSS手段，有很多网站其实是把token放在一个隐藏的input中的，css中有一个input的选择器，可以匹配出input中以某个字符串开头，通过选择器，加载一个外部资源，例如背景图片。但是这有个前提，需要原来的页面存在注入，例如文章里这段代码。

```html
<form action="https://security.love" id="sensitiveForm">
    <input type="hidden" id="secret" name="secret" value="dJ7cwON4BMyQi3Nrq26i">
</form>
<script src="mockingTheBackend.js"></script>
<script>
    var fragment = decodeURIComponent(window.location.href.split("?injection=")[1]);
    var htmlEncode = fragment.replace(/</g,"&lt;").replace(/>/g,"&gt;");
    document.write("<style>" + htmlEncode + "</style>");
</script>
```

从url中的queryString获取参数，插入到dom中，dom中加载样式，来猜解token。

查看该demo的源代码就知道原理了。

[demo](https://security.love/cssInjection/attacker.html)



#### 参考链接

[跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)

[浏览器的同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)

[HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)

[启用了 CORS 的图片](https://developer.mozilla.org/zh-CN/docs/Web/HTML/CORS_enabled_image)

[再见，CSRF：讲解set-cookie中的SameSite属性](https://www.anquanke.com/post/id/83773)

[CSRF 攻击的应对之道](https://www.ibm.com/developerworks/cn/web/1102_niugang_csrf/)

[利用CSS注入（无iFrames）窃取CSRF令牌](http://www.freebuf.com/articles/web/162687.html)

[内容安全策略( CSP )](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)

[XSS 攻击的处理](https://blog.alswl.com/2017/05/xss/)

