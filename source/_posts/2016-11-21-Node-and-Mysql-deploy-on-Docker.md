---
title: Docker开发部署Node小结
date: 2016-11-21 19:22:31
tags: Node Docker
---

在Node部署中，遇到过几个比较坑的地方，也都找到了一些解决方案，这里做一个小结，如果有更好的解决方案，请告诉我;)。

<!-- more -->

# 部署的镜像选择

一开始是直接选择 [Docker Hub](https://hub.docker.com) 的 Node 官方镜像，看了一下 Tag 发现官方大致提供了两种：

`wheezy` 、 `slim` 和 `alpine`

Node.js 7.1版本的镜像大小 [image](https://hub.docker.com/r/library/node/tags/)

```
wheezy 192MB
slim   85MB
latest 258MB

alpine 19MB
```



Dockerfile如下:

``` dockerfile
FROM node:alpine

# 设置相应的时区
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone && \
    apk del tzdata

RUN mkdir -p /opt/workdir

WORKDIR /opt/workdir
COPY . /opt/workdir

RUN npm install --production && npm install pm2 -g

EXPOSE 3000

CMD ["npm", "start"]
```

关于 `alpine` 官方网站 [alpinelinux](https://www.alpinelinux.org/)

官方介绍：

> Alpine Linux is a security-oriented, lightweight Linux distribution based on musl libc and busybox.

单一系统只有4MB左右，很适合我这种强迫症患者

**Note: 如果是直接利用 alpine 构建镜像，所构成的镜像会比官方提供的 alpine 镜像更小，大约小10MB左右(也许是和 alpine 的包管理器最新的 node 版本在6.2有关 )，但并不建议这样做，还是用官方提供的 image 好了😂**

直接使用alpine构建的 `Dockerfile` 如下:

``` dockerfile
FROM alpine

# 设置相应的时区
RUN apk add --no-cache nodejs tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone && \
    apk del tzdata

RUN mkdir -p /opt/workdir

WORKDIR /opt/workdir
COPY . /opt/workdir

RUN npm install --production && npm install pm2 -g

EXPOSE 3000

CMD ["npm", "start"]
```

**Note2号: 这里的 npm start 的代码是 NODE_ENV=production pm2-docker app.js**

关于pm2-docker [pm2-docker文档](http://pm2.keymetrics.io/docs/usage/docker-pm2-nodejs/)

# Docker-Compose 部署 Node & Mysql

## Node 开发环境

因为 `Mysql` 在 `Docker` 环境，所以必须有一个本地测试的数据库配置，这里可以利用环境变量来进行开发环境和生产环境的切换。

[Mysql镜像](https://hub.docker.com/_/mysql/)很方便的提供了 `MYSQL_PORT_3306_TCP_ADDR` 、 `MYSQL_PORT_3306_TCP_PORT` 等环境变量，这样一来就能很方便的写配置文件。

config.js

``` javascript
'use strict'
const path = require('path')

let config = {
    port: 3000,
    staticDir: path.join(__dirname,'..', 'public'),
    mysql: {
        user: 'root',
        pass: process.env.MYSQL_ROOT_PASSWORD,
        host: process.env.MYSQL_PORT_3306_TCP_ADDR,
        port: process.env.MYSQL_PORT_3306_TCP_PORT,
        db: 'yourdb'
    },
    secret: 'yoursecret',
    TOKEN_EXPIRATION: 60 * 60 * 24
}

const local = {
    env: 'local',
    mysql: {
        user: 'root',
        pass: 'pass',
        host: '192.168.33.10',
        port: 3306,
        db: 'yourdb'
    },
    debug: true
}

if (process.env.NODE_ENV === 'development') {
    config = Object.assign(config, local)
}

module.exports = config
```

package.json

``` json
"scripts": {
    "start": "pm2-docker app.js",
    "dev": "NODE_ENV=development nodemon app.js",
    "test": "NODE_ENV=development mocha"
}
```

## Schema 初始化

Mysql 镜像的文档里有这么一段话

> When a container is started for the first time, a new database mysql will be initialized with the provided configuration variables. Furthermore, it will execute files with extensions .sh and .sql that are found in /docker-entrypoint-initdb.d. You can easily populate your mysql services by mounting a SQL dump into that directory and provide custom images with contributed data.

文档中提到了如果在 `docker-entrypoint-initdb.d` 中放上 `.sql` 或 `.sh` 文件，镜像将会自动执行，所以利用 `Docker` 的 `Volume` 就行了。

docker-compose.yml

``` yaml
web:
  image: yourweb:tag
  links:
  - mysql
  ports:
  - 80:3000
  volumes:
  - /opt/workdir/schema.sql:/opt/workdir/db/schema.sql
  environment:
  - MYSQL_ROOT_PASSWORD=yourpass
  depends_on:
  - mysql
mysql:
  image: mysql:latest
  volumes:
  - /opt/workdir/schema.sql:/docker-entrypoint-initdb.d/schema.sql:ro
  - /data/workdir/mysql:/var/lib/mysql
  environment:
  - MYSQL_ROOT_PASSWORD=yourpass
  - MYSQL_DATABASE=db
```

# Docker 持续集成的 Cache 优化

经过几次~~(频繁)~~的修 `BUG` 发现了上述那个 `Dockerfile` 存在一个很严重的问题，每次 `commit` 触发 CI 后，在更新代码的时候都得重新运行一次

`RUN npm install --production && npm install pm2 -g`

然而 `package.json` 并不是频繁变化的，这样使得 `layer` 不能尽可能的被使用，`build` 的时间也变得比较长，经过查资料~~(Google)~~后，找到了解决方案。

Dockerfile

``` dockerfile
FROM node:alpine

RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo "Asia/Shanghai" > /etc/timezone && \
    apk del tzdata

COPY package.json /tmp/package.json

RUN cd /tmp && npm install --production && npm install pm2 -g && \
    mkdir -p /opt/workdir && mv /tmp/node_modules /opt/workdir/

WORKDIR /opt/workdir
COPY . /opt/workdir

EXPOSE 3000

CMD ["npm", "start"]
```

这里的每一个 `RUN` 都对应一个  `layer` 层，这里是先把 `package.json` 先转移到 `/tmp` 目录，在 `/tmp` 里执行 `npm` 的安装，再把 `node_modules` 移回工作目录，最后把代码移进工作目录。

这样做的目的是先进行 `package.json` 的判断，把代码转移放在最后，`package.json` 不变化，`layer` 就不会变化，也就不会有额外的 `layer` 层。

**Note3号: 这里可以直接把package.json直接放到workdir，我这里懒得改了**

# 总结

折腾了一下午，解决了 `Docker` 部署的几个坑。~~顺便拿来凑个数~~
