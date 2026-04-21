---
title: "使用Logrotate归档日志"
description: "aliases:\n/posts/2019-08-15-使用Logrotate归档日志/ 创建tomcat的配置文件 vim /etc/logrotate.d/tomcat\n内容：\n/application/tomcat/logs/catalina.out &#123;dailycopytruncaterotate 30compressnotifemptydateextmissingok&#125; 注意上面内容中大括号前面内容为 待处理日志文件的绝对路径。大括号里面的都是Logrotate工具的参数\n关于每个参数的作用我们可以查看全局文件cat /etc/logrotate.conf 我们可以把配置文件写在/etc/logrotate.conf里面或者放在/etc/logrotate.d下面\n参数解释 daily 表示每天整理一次rotate 20 表示保留20天的备份文件dateext 文件后缀是日期格式,也就是切割后文件是:xxx.log-20171205.gzcopytruncate 用于还在打开中的日志文件，把当前日志备份并截断compress 通过gzip压缩转储以后的日志（gzip -d xxx.gz解压）missingok 如果日志不存在则忽略该警告信息notifempty 如果是空文件的话，不转储#size 5M #当catalina.out大于5M就进行切割，可用可不用！ 不常用的参数 weekly 指定转储周期为每周monthly 指定转储周期为每月nocompress 不需要压缩时，用这个参数nocopytruncate 备份日志文件但是不截断create mode owner group 转储文件，使用指定的文件模式创建新的日志文件nocreate 不建立新的日志文件delaycompress 和 compress 一起使用时，转储的日志文件到下一次转储时才压缩nodelaycompress 覆盖 delaycompress 选项，转储同时压缩errors address 转储时的错误信息发送到指定的Email 地址ifempty 即使是空文件也转储，这个是 logrotate 的缺省选项。mail address 把转储的日志文件发送到指定的E-mail 地址nomail 转储时不发送日志文件olddir directory 转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统noolddir 转储后的日志文件和当前日志文件放在同一个目录prerotate/endscript 在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行postrotate/endscript 在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行 温馨提示：配置文件里一定要配置rotate 文件数目这个参数。如果不配置默认是0个，也就是只允许存在一份日志，刚切分出来的日志会马上被删除\n测试\n调试 （d = debug）参数为配置文件，不指定则执行全局配置文件 logrotate -d /etc/logrotate.d/tomcat.conf\n"
date: 2019-08-15
tags:
  - 运维
  - 日志
---
aliases:

  * /posts/2019-08-15-使用Logrotate归档日志/



## 创建tomcat的配置文件 

vim /etc/logrotate.d/tomcat

内容：
    
    
    /application/tomcat/logs/catalina.out &#123;dailycopytruncaterotate 30compressnotifemptydateextmissingok&#125;
    

注意上面内容中大括号前面内容为 待处理日志文件的绝对路径。大括号里面的都是Logrotate工具的参数

关于每个参数的作用我们可以查看全局文件cat /etc/logrotate.conf 我们可以把配置文件写在/etc/logrotate.conf里面或者放在/etc/logrotate.d下面

## 参数解释 
    
    
    daily 表示每天整理一次rotate 20 表示保留20天的备份文件dateext 文件后缀是日期格式,也就是切割后文件是:xxx.log-20171205.gzcopytruncate 用于还在打开中的日志文件，把当前日志备份并截断compress 通过gzip压缩转储以后的日志（gzip -d xxx.gz解压）missingok 如果日志不存在则忽略该警告信息notifempty 如果是空文件的话，不转储#size 5M #当catalina.out大于5M就进行切割，可用可不用！
    

## 不常用的参数 
    
    
    weekly 指定转储周期为每周monthly 指定转储周期为每月nocompress 不需要压缩时，用这个参数nocopytruncate 备份日志文件但是不截断create mode owner group 转储文件，使用指定的文件模式创建新的日志文件nocreate 不建立新的日志文件delaycompress 和 compress 一起使用时，转储的日志文件到下一次转储时才压缩nodelaycompress 覆盖 delaycompress 选项，转储同时压缩errors address 转储时的错误信息发送到指定的Email 地址ifempty 即使是空文件也转储，这个是 logrotate 的缺省选项。mail address 把转储的日志文件发送到指定的E-mail 地址nomail 转储时不发送日志文件olddir directory 转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统noolddir 转储后的日志文件和当前日志文件放在同一个目录prerotate/endscript 在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行postrotate/endscript 在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行
    

温馨提示：配置文件里一定要配置rotate 文件数目这个参数。如果不配置默认是0个，也就是只允许存在一份日志，刚切分出来的日志会马上被删除

测试

  * 调试 （d = debug）参数为配置文件，不指定则执行全局配置文件



logrotate -d /etc/logrotate.d/tomcat.conf

  * 强制执行（-f = force），可以配合-v(-v =verbose）使用，注意调试信息默认携带-v；



logrotate -v -f /etc/logrotate.d/tomcat.conf

配合crontab定时器即可实现每天备份当天日志并压缩归档了。