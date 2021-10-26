---
title:      "Supervisor自启动Flask项目"
date:       2021-10-25 20:35:50
tags:
     - 19级
     - Flask
     - Linux
author:     "俞润鼎"
---


# Supervisor自启动Flask项目

# 一、概述
### 官方介绍
> Supervisor is a client/server system that allows its users to monitor and control a number of processes on UNIX-like operating systems.


### 我的介绍
> 开机启动进程，自动重启崩溃进程，看看进程状态(不用写shell脚本)<br>

#### 样例环境
> 这里介绍启动flask+gunicorn+mysql+redis+nginx项目，默认该部分正常运行
> 系统centos7.6

# 二、安装
```shell script
pip install supervisor
```
如果没有自动配置环境变量成功，需加上

# 三、启动
### 1.启动
```shell script
supervisord -c /etc/supervisord.conf # 带配置文件的启动

```
### 2.设置开机自启动
输入
```shell script
vim /usr/lib/systemd/system/supervisord.service
```
填入下列内容
```ini

[Unit]
Description=Supervisor daemon

[Service]
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf # 启动命令
ExecStop=/usr/bin/supervisorctl shutdown    # 停止程序的命令
ExecReload=/usr/bin/supervisorctl reload     # 重启进程的命令
KillMode=process   
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target

```
# 四、相关指令
```shell script
service supervisord start #启动,默认配置文件
service supervisord stop #停止
service supervisord status #状态
supervisorctl shutdown #关闭所有任务
supervisorctl stop|start program_name #启动或停止服务
supervisorctl status #查看所有任务状态
```
# 五、配置文件
### 1.supervisord自己的配置文件
如果没有则创建 /ect/supervisord.conf 在末尾加入或修改
```ini
[include]
files = /etc/supervisord/conf.d/*.conf
```
然后创建文件夹/ect/supervisord/conf.d<br>
之后就可以在此路径下放入需要supervisord管理的进程的配置文件<br>
启动supervisord的时候，他会读取该路径下所以.conf文件<br>
（当然也可以把下面的文件内容直接加在supervisord.conf末尾）

参考配置文件
```ini
;Sample supervisor config file.

[unix_http_server]
file=/run/supervisor/supervisor.sock   ; (the path to the socket file)
;chmod=0700                 ; sockef file mode (default 0700)
;chown=nobody:nogroup       ; socket file uid:gid owner
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

;[inet_http_server]         ; inet (TCP) server disabled by default
;port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for all iface)
;username=user              ; (default is no username (open server))
;password=123               ; (default is no password (open server))

[supervisord]
logfile=/var/log/supervisor/supervisord.log  ; (main log file;default $CWD/supervisord.log)
logfile_maxbytes=50MB       ; (max main logfile bytes b4 rotation;default 50MB)
logfile_backups=10          ; (num of main logfile rotation backups;default 10)
loglevel=info               ; (log level;default info; others: debug,warn,trace)
pidfile=/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
nodaemon=false              ; (start in foreground if true;default false)
minfds=1024                 ; (min. avail startup file descriptors;default 1024)
minprocs=200                ; (min. avail process descriptors;default 200)
;umask=022                  ; (process file creation umask;default 022)
;user=root                 ; (default is current user, required if root)
;identifier=supervisor       ; (supervisord identifier, default is 'supervisor')
;directory=/tmp              ; (default is not to cd during start)
;nocleanup=true              ; (don't clean up tempfiles at start;default false)
;childlogdir=/tmp            ; ('AUTO' child log dir, default $TEMP)
;environment=KEY=value       ; (key value pairs to add to environment)
;strip_ansi=false            ; (strip ansi escape codes in logs; def. false)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///run/supervisor/supervisor.sock ; use a unix:// URL  for a unix socket
;serverurl=http://0.0.0.0:9001 ; use an http:// url to specify an inet socket
;username=yrd              ; should be same as http_username if set
;password=00929.                ; should be same as http_password if set
;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
;history_file=~/.sc_history  ; use readline history if available

; The below sample program section shows all possible program subsection values,
; create one or more 'real' program: sections to be able to control them under
; supervisor.

;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;autorestart=true              ; retstart at unexpected quit (default: true)
;startsecs=10                  ; number of secs prog must stay running (def. 1)
;startretries=3                ; max # of serial start failures (default 3)
;exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A=1,B=2           ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)

; The below sample eventlistener section shows all possible
; eventlistener subsection values, create one or more 'real'
; eventlistener: sections to be able to handle event notifications
; sent by supervisor.

;[eventlistener:theeventlistenername]
;command=/bin/eventlistener    ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;events=EVENT                  ; event notif. types to subscribe to (req'd)
;buffer_size=10                ; event buffer queue size (default 10)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=-1                   ; the relative start priority (default -1)
;autostart=true                ; start at supervisord start (default: true)
;autorestart=unexpected        ; restart at unexpected quit (default: unexpected)
;startsecs=10                  ; number of secs prog must stay running (def. 1)
;startretries=3                ; max # of serial start failures (default 3)
;exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (default 10)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups        ; # of stderr logfile backups (default 10)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A=1,B=2           ; process environment additions
;serverurl=AUTO                ; override serverurl computation (childutils)

; The below sample group section shows all possible group values,
; create one or more 'real' group: sections to create "heterogeneous"
; process groups.

;[group:thegroupname]
;programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
;priority=999                  ; the relative start priority (default 999)

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisord/conf.d/*.conf
```

> [inet_http_server] 下的配置可以获得supervisord的网页管理页面
![参考](./f1.png)
### 2.其他进程的配置文件
完整配置解释可以查看[这里](http://supervisord.org/configuration.html)
> flask+gunicorn
```ini
[program:blog]  ; 进程名称，用supervisorctl控制时使用的名称
directory=/home/blog    ; 执行路径，后续指令会在该路径下执行
command=/home/blog/venv/bin/gunicorn -c /home/blog/gun_config.py manage:app ; 带虚拟环境的启动指令

autostart=true  ; 自启动
autorestart=true    ; 自动重启(如果异常关闭)
startsecs=10    ; 判断为启动成功需要正常运行的时间

priority=5  ; 优先级，可以用来控制启动顺序

user=root   ; 执行者角色

; 两个日志输出路径
stderr_logfile=/home/blog/log/blog_stderr.log
stdout_logfile=/home/blog/log/blog_stdout.log

; 日志相关设置
redirect_stderr=true

stdout_logfile_maxbytes=20MB

stdout_logfile_backups=5
```


> mysql
```ini
[program:mysql] 
directory = /usr/local/mysql/bin            ; 
command=/usr/bin/pidproxy /data/mysql/mysql.pid /usr/local/mysql/bin/mysqld_safe     ;要作为后台进程启动，不然无法识别启动成功
autostart=true
autorestart=true
startsecs=10

priority=4
user=root

stderr_logfile=/home/blog/log/blog_stderr.log
stdout_logfile=/home/blog/log/blog_stdout.log

redirect_stderr=true

stdout_logfile_maxbytes=20MB

stdout_logfile_backups=5
```
> redis
```ini
[program:redis] 
directory = /usr/local/          
command=redis-server /usr/local/redis-6.2.5/redis.conf
autostart=true
autorestart=true
startsecs=10

priority=6
user=root
daemonize=no

stderr_logfile=/home/blog/log/blog_stderr.log
stdout_logfile=/home/blog/log/blog_stdout.log

redirect_stderr=true

stdout_logfile_maxbytes=20MB

stdout_logfile_backups=5

```

> nginx
```ini
[program:nginx] 
directory = /usr/local/nginx/sbin           
command=/usr/local/nginx/sbin/nginx -g 'daemon off
autostart=true
autorestart=true
startsecs=10

priority=4
user=root

stderr_logfile=/home/blog/log/blog_stderr.log
stdout_logfile=/home/blog/log/blog_stdout.log

redirect_stderr=true

stdout_logfile_maxbytes=20MB

stdout_logfile_backups=5
```
