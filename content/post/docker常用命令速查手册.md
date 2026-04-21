---
title: "Docker 常用命令速查手册"
description: "Docker 命令太多记不住？这篇速查手册涵盖镜像、容器、网络、数据卷的日常操作，收藏备用。"
date: 2026-03-25
tags:
  - Docker
  - 运维
---
> Docker 命令又多又杂，这篇按使用场景整理，覆盖 90% 的日常操作。建议收藏，用到时直接查。

* * *

## 一、镜像管理 
    
    
    # 搜索镜像
    docker search nginx
    
    # 拉取镜像
    docker pull nginx:latest
    docker pull python:3.11-slim
    
    # 查看本地镜像
    docker images
    docker image ls
    
    # 删除镜像
    docker rmi nginx:latest
    
    # 清理悬空镜像（无标签的）
    docker image prune
    
    # 查看镜像详情（层信息、大小）
    docker history nginx:latest
    

* * *

## 二、容器生命周期 
    
    
    # 创建并运行容器
    docker run -d --name my-nginx -p 80:80 nginx
    
    # -d 后台运行
    # --name 命名
    # -p 端口映射（宿主机:容器）
    # -v 数据卷挂载
    # -e 环境变量
    # --restart 重启策略
    
    # 查看运行中的容器
    docker ps
    
    # 查看所有容器（包括已停止的）
    docker ps -a
    
    # 启动 / 停止 / 重启
    docker start my-nginx
    docker stop my-nginx
    docker restart my-nginx
    
    # 删除容器
    docker rm my-nginx
    
    # 强制删除运行中的容器
    docker rm -f my-nginx
    
    # 进入容器
    docker exec -it my-nginx /bin/bash
    docker exec -it my-nginx /bin/sh    # Alpine 镜像用 sh
    
    # 查看容器日志
    docker logs my-nginx
    docker logs -f my-nginx       # 实时跟踪
    docker logs --tail 100 my-nginx  # 最后100行
    
    # 查看容器资源占用
    docker stats
    docker stats my-nginx
    
    # 查看容器详情（IP、挂载等）
    docker inspect my-nginx
    

* * *

## 三、docker run 常用参数 
    
    
    docker run -d \
      --name myapp \
      -p 8080:80 \
      -v /data/app:/app/data \
      -e TZ=Asia/Shanghai \
      -e MYSQL_ROOT_PASSWORD=123456 \
      --restart unless-stopped \
      --network my-net \
      nginx
    

参数 | 说明 | 示例  
---|---|---  
`-d` | 后台运行 |   
`-p` | 端口映射 | `-p 8080:80`  
`-v` | 挂载数据卷 | `-v /host/path:/container/path`  
`-e` | 环境变量 | `-e MYSQL_ROOT_PASSWORD=xxx`  
`--name` | 容器名称 | `--name myapp`  
`--restart` | 重启策略 | `no / on-failure / always / unless-stopped`  
`--network` | 指定网络 | `--network my-net`  
`-v $(pwd):/app` | 挂载当前目录 |   
`--rm` | 容器停止后自动删除 | 适合一次性任务  
  
* * *

## 四、数据卷管理 
    
    
    # 创建数据卷
    docker volume create mydata
    
    # 查看数据卷列表
    docker volume ls
    
    # 查看数据卷详情
    docker volume inspect mydata
    
    # 删除数据卷
    docker volume rm mydata
    
    # 清理未使用的数据卷
    docker volume prune
    

> 持久化数据一定要用 `-v` 挂载，否则容器删除后数据就丢了。

* * *

## 五、网络管理 
    
    
    # 创建网络
    docker network create my-net
    
    # 查看网络列表
    docker network ls
    
    # 容器加入网络（同一网络内可用容器名互访）
    docker network connect my-net my-nginx
    
    # 断开网络
    docker network disconnect my-net my-nginx
    
    # 删除网络
    docker network rm my-net
    

* * *

## 六、Docker Compose 
    
    
    # 启动所有服务
    docker compose up -d
    
    # 停止所有服务
    docker compose down
    
    # 停止并删除数据卷
    docker compose down -v
    
    # 查看服务状态
    docker compose ps
    
    # 查看日志
    docker compose logs -f
    
    # 重启某个服务
    docker compose restart nginx
    
    # 重新构建并启动
    docker compose up -d --build
    

### docker-compose.yml 示例 
    
    
    version: '3.8'
    services:
      nginx:
        image: nginx:latest
        ports:
          - "80:80"
        volumes:
          - ./html:/usr/share/nginx/html
        restart: unless-stopped
    
      mysql:
        image: mysql:8.0
        environment:
          MYSQL_ROOT_PASSWORD: 123456
          MYSQL_DATABASE: myapp
        volumes:
          - mysql-data:/var/lib/mysql
        restart: unless-stopped
    
    volumes:
      mysql-data:
    

* * *

## 七、镜像构建 

### Dockerfile 示例 
    
    
    FROM python:3.11-slim
    
    WORKDIR /app
    
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt
    
    COPY . .
    
    EXPOSE 8000
    
    CMD ["python", "app.py"]
    

### 构建与推送 
    
    
    # 构建镜像
    docker build -t myapp:1.0 .
    
    # 查看构建历史
    docker history myapp:1.0
    
    # 打标签
    docker tag myapp:1.0 myregistry/myapp:1.0
    
    # 推送到仓库
    docker push myregistry/myapp:1.0
    

* * *

## 八、清理命令 
    
    
    # 一键清理（停止的容器、未使用的网络、悬空镜像、构建缓存）
    docker system prune
    
    # 深度清理（包括未使用的数据卷）⚠️ 慎用
    docker system prune -a --volumes
    
    # 查看磁盘占用
    docker system df
    

* * *

## 九、实用技巧 

### 查看容器内进程 
    
    
    docker top my-nginx
    

### 在容器和宿主机之间复制文件 
    
    
    # 从容器复制到宿主机
    docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf
    
    # 从宿主机复制到容器
    docker cp ./nginx.conf my-nginx:/etc/nginx/nginx.conf
    

### 查看容器端口映射 
    
    
    docker port my-nginx
    

### 导出和导入镜像 
    
    
    docker save -o myapp.tar myapp:1.0
    docker load -i myapp.tar
    

### 查看容器实时资源 
    
    
    docker stats --no-stream    # 只显示一次
    docker stats my-nginx       # 指定容器
    

* * *

_收藏这篇，Docker 命令不用死记，用到的时候来查就行。_