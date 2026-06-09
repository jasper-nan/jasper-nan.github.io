---
title: "Linux 进程管理实战：从查找、过滤到终止"
description: "日常开发中经常需要管理服务器进程，这篇文章覆盖 ps、grep、awk、kill、nohup 的实战用法。"
date: 2026-03-25
tags:
  - Linux
  - 运维
---
> 在服务器上部署应用时，进程管理是最基础也最常用的操作。本文整理了日常开发中最高频的几个命令。

* * *

## 一、查找进程：ps 

`ps`（Process Status）是最常用的进程查看命令。
    
    
    # 列出所有进程的详细信息
    ps -ef
    

输出字段说明：

字段 | 含义  
---|---  
UID | 进程所属用户  
PID | 进程号  
PPID | 父进程号  
STIME | 启动时间  
CMD | 启动命令  
  
### 常用组合 
    
    
    # 查看所有进程（BSD 风格，更直观）
    ps aux
    
    # 按内存使用排序（Top 10）
    ps aux --sort=-%mem | head -10
    
    # 按 CPU 使用排序（Top 10）
    ps aux --sort=-%cpu | head -10
    

* * *

## 二、过滤进程：grep 

在进程列表中精确匹配目标：
    
    
    ps -ef | grep "关键字"
    

> ⚠️ 注意：`grep` 本身也会出现在结果中，因为命令行参数中包含"关键字"。

**过滤掉 grep 自身：**
    
    
    ps -ef | grep "java" | grep -v grep
    

* * *

## 三、提取进程号：awk 

拿到 PID 后经常需要做进一步操作（比如 kill），用 `awk` 提取：
    
    
    ps -ef | grep "java" | grep -v grep | awk '{print $2}'
    

  * `awk '{print $2}'`：提取每行的第二个字段，即 PID



* * *

## 四、终止进程：kill 

### 优雅终止 
    
    
    kill PID
    

发送 `SIGTERM`（15）信号，进程可以自行清理资源后退出。

### 强制终止 
    
    
    kill -9 PID
    

发送 `SIGKILL`（9）信号，进程无法捕获，立即被系统终止。**慎用** ，可能导致数据丢失。

### 终止一组进程 
    
    
    # 终止所有 java 进程
    kill -9 $(ps -ef | grep "java" | grep -v grep | awk '{print $2}')
    

* * *

## 五、程序后台运行：nohup 

SSH 登录服务器启动程序后，关闭终端进程就死了。用 `nohup` 解决：

### 基本格式 
    
    
    nohup <command> > output.log 2>&1 &
    

拆解说明：

部分 | 作用  
---|---  
`nohup` | 防止 SIGHUP 信号杀死进程  
`> output.log` | 标准输出重定向到日志文件  
`2>&1` | 标准错误合并到标准输出  
`&` | 放入后台运行  
  
### 实际示例 
    
    
    # 后台运行 MySQL
    nohup mysql --user=root --password=my_pass > mysql.log 2>&1 &
    
    # 后台运行 Python 脚本
    nohup python3 app.py > app.log 2>&1 &
    
    # 后台运行 Java 应用
    nohup java -jar myapp.jar > myapp.log 2>&1 &
    

### 查看 nohup 启动的进程 
    
    
    ps aux | grep "app.py"
    

### 查看输出日志 
    
    
    tail -f output.log
    

* * *

## 六、日志分析：Goaccess 

用 Goaccess 生成 Nginx 访问日志的可视化报告：

### 实时 HTML 报告 
    
    
    goaccess /var/log/nginx/access.log \
      -o /usr/local/nginx/html/report.html \
      --real-time-html \
      --time-format='%H:%M:%S' \
      --date-format='%d/%b/%Y' \
      --log-format=COMBINED
    

### 自定义日志格式 

如果你的 Nginx 日志格式不是标准 COMBINED，可以用正则自定义：
    
    
    goaccess access.log \
      -o /usr/local/nginx/html/report.html \
      --log-format='~h{, } %^[%d:%t %^] "%r" %s %b "%R" "%u"' \
      --date-format=%d/%b/%Y \
      --time-format=%T
    

  * `~h{, }`：匹配可选协议头
  * `%d:%t`：自定义日期时间解析
  * `"%r"`：匹配原始请求行



* * *

## 📋 速查表 
    
    
    # 查找进程
    ps -ef | grep "关键词" | grep -v grep
    
    # 提取 PID
    ps -ef | grep "关键词" | grep -v grep | awk '{print $2}'
    
    # 终止进程
    kill PID          # 优雅终止
    kill -9 PID       # 强制终止
    
    # 后台运行
    nohup command > output.log 2>&1 &
    
    # 查看日志
    tail -f output.log
    

* * *

_希望这篇文章对你有帮助！如果觉得有用，欢迎分享给需要的朋友。_