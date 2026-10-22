# 一小时速通Docker

## 一、Docker基础概念

#### 1、Docker是什么？解决了什么问题？

Docker是什么？

Docker是一个容器化平台。它可以把应用程序和所需的依赖打包成Docker 镜像，然后通过容器读取你打包镜像的内容，将项目跑起来。

Docker技术解决了环境一致性的问题（在我机器上是好的）。

总结：Docker是一种容器化技术，它把应用和依赖打包成标准镜像（模板），让应用可以在不同环境中以相同的方式运行，主要决定“环境不一致”和“部署复杂”的问题

#### 2、Docker和虚拟机（VM）有什么区别？

#### ![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker1-1.png)

Docker把应用程序和依赖打包成镜像，然后以容器形式运行。

一个 容器通常包含：

1）应用代码

2）依赖库

3）配置文件

4）环境变量

5）运行时

容器不包含完整的操作系统内核。它会共享主机的操作系统内核（OS kernel）

而虚拟机（VM）都有自己的完整操作系统

总结：Docker是操作系统级虚拟化，多个容器共享宿主机内核；虚拟机是硬件级虚拟化，每个VM都有自己的完整操作系统，所以Docker更轻量、启动更快，VM隔离更彻底。

#### 3、Docker的核心组件有哪些？

#### ![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker1-3.png)

一句话讲：Docker采用C/S架构，核心组件主要包括

1）Docker Client：命令行客户端（docker命令）

2）Docker Daemon（docked）：服务端守护进程，管理镜像、容器、网络、存储

3）Docker Registry：镜像仓库（Docker Hub、私有仓库）

4）Docker Host：运行Docker的宿主机

Docker的运行流程：

```
Docker run -> Client发请求 -> Daemonn接收 ->本地有镜像直接运行 
->没有从镜像仓库拉取 -> 创建容器运行
```

* 注意：

Client和Daemon可以在不同机器上（运程Docker）

Daemon是核心，所有操作都经过它

Registry存镜像，类似代码的git仓库

#### 4、镜像、容器、仓库的关系？

#### ![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker1-4.png)

总结：三者关系：镜像（Image）-->容器（Container）-->仓库（Registry）

> 镜像（Image）：只读模板，包含运行应用所需的一切（代码、运行时、库、配置）
>
> 容器（Container）：镜像的运行实例，一个镜像可创建多个容器
>
> 仓库（Resgistry）：存储和分发镜像的地方 

类比：

> 镜像 = 类（class）
>
> 容器 = 实例（object）
>
> 仓库 = 镜像商店

```linux
# 运行容器（-d 后台运行，-p 映射端口）
docker run -d nginx
# 拉取镜像
docker pull
# 查看运行中的容器
docker ps
#构建镜像（基于当前目录的Dockerfile）
docker build
#将容器保存为镜像
docker commit <container>
```

* 注意

镜像是只读的，容器在镜像上加一层可写层

一个镜像可启动多个独立容器

镜像分层存储，共享层节省空间

#### 5、Docker镜像分层原理

#### ![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker1-5.png)

总结：Docker镜像由多层只读层组成，每层代表Dockerfile的一条指令，层与层叠加构成最终镜像

特点：

> 分层存储：每个指令生成一层
>
> 共享层：多个镜像可共享相同的基础层，节省空间和传输
>
> 写时复制：（Copy-on-Write）：容器运行时在顶层加可写层，修改时复制对应层

好处：

> 镜像构建快（缓存层，未改变的层不重建）
>
> 存储省（共享公共层）
>
> 传输快（只传变化的层）

要点：

> Dockerfile每条指令产生一层，层数影响构建效率
>
> 合并指令（用&&连接RUN）减少层数
>
> 容器的写操作只影响可写层，不改变镜像
>
> docker history 查看镜像分层

* 一句话说：镜像= 多层只读层叠加，共享层省空间，容器加可读层+写时复制，Docker每条指令一层

## 二、Dockerfile 与镜像构建

### 1、Docker常用指令有哪些？



|    指令    |           作用           |              示例              |
| :--------: | :----------------------: | :----------------------------: |
|    RUN     |      构建时执行命令      |      RUN apt install curl      |
|    COPY    |      复制文件到镜像      |       COPY app.jar /app/       |
|    ADD     | 复制文件（支持URL/解压） |         ADD url /app/          |
|  WORKDIR   |       设置工作目录       |          WORKDIR /app          |
|    ENV     |       设置环境变量       |      ENV NODE_ENV = prod       |
|   EXPOSE   |         声明端口         |          EXPOSE 8080           |
|    CMD     | 容器启动命令（可被覆盖） | CMD ["java"，"-jar","app,jar"] |
| ENTRYPOINT | 容器启动入口（不易覆盖） |      ENTERPOINT ["java"]       |
|   VOLUE    |        声明挂载点        |          VOLUME /data          |
|    ARG     |         构建参数         |       ARG VERSION = 1.0        |
|    FORM    |         基础镜像         |       FROM ubuntu：22.04       |

要点：

RUN在构建时执行，CMD/ENTERPOINT 在运行时执行

ENTERPOINT 不易被docker run覆盖，CMD可以

COPY优先于ADD（ADD 有解压等副作用）

每个RUN产生一层，用&&合并减少层数

* 一句话：FROM定基础、RUN构建时运行、COPY复制文件、CMD/ENTERPOINT运行时执行，合并RUN减层数

### 2、CMD和ENTERPOINT区别？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker2-2.png)

| 对比 |         CMD          |          ENTERPOINT           |
| :--: | :------------------: | :---------------------------: |
| 作用 |   容器默认启动命令   |         容器固定命令          |
| 覆盖 | docker run参数可覆盖 |     需 --entrypoint 覆盖      |
| 用法 |  CMD["cmd"，"arg"]   | ENTERPOINT["cmd"]+ CMD[“arg”] |
| 场景 |   灵活指定运行命令   |     固定入口 + 可追加参数     |

组合用法：

```
ENTEROPINT ["java","-jar","app.jar"]
CMD ["--server.port = 8080"]
```

ENTERPOINT固定入口

CMD提供默认参数，可被run时覆盖

要点：

两者都有exec形式（json数组）和 shell形式

推荐exec形式，shell形式会作为 /bin/sh -c的参数

只有最后一条CMD/ENTERPOINT 生效

* 一句话：CMD可被run参数覆盖，ENTERPOINT固定入口不易覆盖；推荐ENTERPOINT固定入口 + CMD默认参数

### 3、怎么优化Dockerfile？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker2-3.png)

总结：Docker file 优化目标是镜像更小、构建更快、更安全

1）减小镜像体积：

- 用精简基础镜像（alpine、distroless、scratch）

- 多阶段构建（multi-stage build），只保留编译产物
- 清理包管理器缓存（apt-get clean、rm -rf /var/lib/apt/lists）

2）加速构建

- 合理利用层缓存：把不变的指令放前面（依赖安装），变化的放后面（代码）
- 合并RUN指令减少层数
- .dockerignore 排除无关文件

3）安全性：

- 不用root用户运行（创建普通用户 USER）
- 扫描镜像漏洞（trivy）

多阶段构建示例：

```
# 构建阶段
FROM maven:3.8 AS build
COPY ./app
WORKDIR /app
RUN mvn clean package

# 运行阶段
FROM openjdk：11 -jre-slim
COPY --from=build /app/target/app.jar  /app.jar
ENTERPOINT ["java","-jar","/app.jar"]
```

要点：

多阶段构建最终镜像只含运行阶段，不含编译工具

alpine镜像小但可能缺glibc，注意兼容性

distroless比alpine 更小更安全

* 一句话：Dockerfile优化 = 精简基础镜像 +多阶段构建 + 清理缓存 +层缓存利用 +非root用户，目标是小快安全

### 4、COPY和ADD区别

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker2-4.png)

|     对比     |  COPY  |     ADD     |
| :----------: | :----: | :---------: |
| 复制本地文件 |  支持  |    支持     |
|   URL下载    | 不支持 |    支持     |
|   自动解压   | 不支持 | 支持（tar） |
|    推荐度    |  推荐  |  尽量不用   |

要点：

ADD能自动解压tar包，能从URL下载文件

但ADD的自动解压可能有意外行为

官方推荐优先用COPY，ADD只在需要解压时用

URL下载建议用RUN curl /wget （能缓存和清理）

* 一句话：COPY 只复制文件，ADD 多了解压和URL下载 ，推荐用COPY ,ADD仅在解压使用

## 三、容器运行与存储

### 1、容器的生命周期状态？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker3-1.png)

核心回答：

Docker容器以下状态：

- created：已创建但未启动
- running：运行中
- paused：已暂停
- stopped：已停止
- restarting：重启中
- exited：已退出
- dead：已死亡（无法启动）

状态转换：

- created ——> start ——>running
- running ——> stop ——>stopped
- stopped ——> start ——> running
- running ——> pause ——> paused ——> unpause ——> running
- running ——>  kill ——> exited

要点：

  docker ps 看运行中的，docker ps -a 看所有

stopped 和 exited 区别：stop优雅停止 ，exited是进程停止

* 一句话：容器状态 = created/running/paused/stopped/exited/dead，ps看运行中的，ps -a 看所有的

### 2、怎么进入容器和复制文件内

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker3-2.png)

```
# 进入容器
docker exec -it <container> /bin/bash # 进入运行中的容器
docker attach <container> #附加到容器主进程

# 复制文件
docker cp local_file container：/path  # 本地->容器
docker cp container：/path local_file  # 容器->本地

# 查看日志
docker logs -f <container>   # 实时查看日志
docker logs --tail 100 <container>   # 查看最后100行日志
```

要点：

exec 是开新进程容器，attach是连接到主进程

cp 可以在容器和宿主机之间复制文件

日志看 stdout/stderr，应用日志要打到标准输出

容器内没有sstd，用exec进入

* 一句话：进容器用exec -it ，复制文件用docker cp ，日志用docker logs -f ，容器内一般没有sshd

### 3、Docker数据卷（Volume）是什么？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker3-3.png)

总结：Volume是Docker持久化数据的机制，把宿主机目录或Docker管理的卷挂载到容器，容器删除数据不丢

三种挂载方式：

1）命名卷（Named Volume）：Docker管理，docker volume create

2）绑定挂载（Bind Mount）:直接挂载宿主机目录

3）tmpfs 挂载：在存在内存中

```
# 命名卷
docker run -v mydata:/data nginx

# 绑定挂载
docker run -v /host/path:/data nginx

# 查看卷
docker volume ls
```

要点：

Volume是容器持久化的唯一方式（容器层删除数据丢失）

命名卷由Docker管理，跨平台

绑定挂载直接用宿主机路径，灵活但依赖宿主机

数据库等有状态服务必须用Volume

* 一句话讲：Volume实现数据持久化，命名卷Docker管理，绑定挂载用宿主机路径，有状态服务必须用。

### 4、怎么限制容器资源？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker3-4.png)

核心回答：Docker可以限制容器的CPU、内存等资源。

```
# 内存限制
docker run --memory=“512m” nginx   # 最大内存 512MB
docker run --memory=“512m” --memory-swap= “1g” nginx   # 内存+交换 1GB

# CPU限制
docker run --cpus=“1.5” nginx  #最多用1.5核
docker run --cpuset-cpus=“0-2” nginx  #只用0-2号核
docker run --cpu-shares=1024 nginx  #CPU相对权重

# 磁盘限制
docker run --storage-opt size=10G nginx  #容器层最大 10GB
```

要点：

- --memory 限制内存，超过会被OOM kill
- --cpus 限制核心数 （Docker 1.13+）
- --cpu-shares 是相对权重，资源紧张时生效
- 生产环境必须设置资源限制，防止容器吃光宿主机资源

* 一句话：--memory限内存，--cpus 限核心数、--cpu-shares 绑核 ，生产必须设资源限制防止宿主机资源被吃光。

## 四、网络

### 1、Docker网络模式有哪些？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker4-1.png)

总结：

Docker有5中网络模式：

1）bridge（默认）：容器通过docker0 网桥通信，有独立IP

2）host：容器共享宿主机网络命名空间，直接用宿主机IP

3）container：和另一个容器共享网络命名空间

4）none：无网络，只有Io

5）自定义网络：用户创建的bridge/overlay网络

```
# 查看网络
docker metwork ls

# 创建自定义网络
docker network create my-net

# 容器加入网络
docker run --network my-net nginx

# host模式
docker run --network host nginx
```

要点：

bridge模式下容器间可用容器名通信（自定义网络支持DNS）

host模式性能好但端口冲突，且不安全

不同网络的容器默认隔离，需要加入同一网络才能通信

overlay网络用于跨主机容器通信（Docker Swarm）

* 一句话讲：Docker网络 = bridge默认桥接/host共享宿主机/container共享容器/none无网络/自定义，自定义 bridge 支持容器名DNS

### 2、容器间怎么通信？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker4-2.png)

核心回答：

容器间通信主要方式：

1）同一自定义网络：用容器名作为hostname通信（Docker DNS）

```
docker network create my-net
docker run -d --network my-net --name web nginx
docker run -d --network my-net --name app myapp
# app 容器内可通过http://web  访问web 容器
```

2）--link（已过时）：连接两个容器

3）host网络：都用宿主机网络，通过端口通信

4）端口映射：把容器端口映射到宿主机，通过宿主机IP通信

要点：

推荐用自定义网络 + 容器名通信

同一bridge网络的容器互通，不同网络隔离

生产用Docker Compose/K8s 管理网络更方便

* 一句话：容器通信 = 同一自定义网络用容器名DNS通信（推荐），或端口映射通过宿主机，--link已过时。

### 3、EXPOSE 和 -p端口映射区别？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker4-3.png)

总结：

EXPOSE：Dockerfile中声明容器监听的端口，只是文档说明，不实际映射

-p （publish）：docker run时容器端口映射到宿主机端口

```
EXPOSE 8080  #声明容器监听 8080 ，不映射
```

```
docker run -p 8080：80 nginx   # 宿主机 8080 -> 容器 80
docker run -p nginx            #自动映射所有EXPOSE的端口到随机宿主机端口
docker run --network host nginx  #直接用宿主机网络
```

要点：

EXPOSE只是声明，不映射；-p 才实际映射

-p 格式：宿主机端口：容器端口

-p 自动映射所有EXPOSE端口到宿主机随机端口

不映射端口，外部无法直接访问容器

* 一句话：EXPOSE只是声明端口不映射，-p才实际映射（宿主机：容器），-p自动映射所有EXPOSE端口。

## 五、Docker Compose与编排

### 1、Docker Compose是什么？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker5-1.png)

总结 ：Docker Compose 是定义和运行多容器应用的工具，用YAML文件配置多个容器，一条命令启动全部。

docker-compose.yml示例：

```
version:"3"
services:
  web:
   image:nginx
   port:
     -"80:80"
  db:
   image:mysql
   environment:
       MYSQL_ROOT_PASSWORD:secret
   volumes:
       -db-data:/var/lib/mysql
volumes:
   db-data:
```

常用命令：

```
docker compose up -d   #后台启动所有服务
docker compose down    #停止并删除
docker compose ps    #查看服务状态
docker compose logs -f    #查看日志
docker compose build    构建镜像
```

要点：

Compose用一个文件管理多个容器的依赖、网络、卷

适合单机多容器应用

生产环境多机用K8s，单机用Compose足够

Compose 自动为服务创建默认网络，服务间可用服务名通信

* 一句话：Compose = YAML定义多容器应用，up一键启动，down一键销毁，适合单机多容器编排。

## 六、安全 与运维

### 1、Docker怎么保证安全？

总结：Docker安全从镜像、容器、宿主、网络四方面保障

1）镜像安全:

> 用可信基础镜像（官方镜像）
>
> 扫描漏洞（trivy、docker scan）
>
> 不用latest标签，用固定版本

2)容器安全:

>不用root 用户（USER 指令）
>
>限制容器能力（--cap-drop 去掉不必要权限）
>
>设置只读根文件系统（--read-only）
>
>限制资源（CPU、内存）

3)宿主机安全

>定期更新 Docker 和内核
>
>限制 Docker daemon 权限
>
>审计 Docker 操作

4)网络安全:

>不暴露不必要的端口
>
> 用自定义网络隔离
>
>不使用 --privileged

要点：

- --privileged 给容器几乎所有权限，非常危险
- 不要把宿主机根目录挂载到容器
- 生产环境必须做镜像漏洞扫描

* 一句话：Docker安全=可信镜像+非root用户+最小权限+资源限制+网络隔离，禁用--privileged。

### 2、怎么排查容器问题？

![](https://prosonal-document-1505077281.cos.ap-guangzhou.myqcloud.com/Docker/Docker6-2.png)

核心回答：

容器问题排查步骤：

1. 看状态: docker ps -a 看容器状态和退出码
2. 看日志: docker logs  看应用日志
3. 看详情: docker inspect  看配置
4. 进容器: docker exec -it  sh 内部排查
5. 看资源: docker stats 看CPU/内存

常见问题：

- 退出码 137: 被 OOM kill，内存不足
- 退出码 143: 收到 SIGTERM (docker stop)
- 端口冲突: 宿主机端口已被占用
- 镜像拉取失败: 网络或仓库权限问题

要点：

logs 是最常用的排查手段

容器立即退出多半是启动命令错误或应用崩溃

 inspect 可以看环境变量、挂载、网络等配置

* 一句话记: 容器排查=ps看状态/退出码→logs看日志→inspect看配置→exec进容器，137是OOM、143是stop。

## 七 、Docker常用命令

```
# 镜像
docker images                # 列出镜像
docker pull nginx            # 拉取镜像
docker rmi nginx             # 删除镜像
docker build -t myapp .      # 构建镜像

# 容器
docker run -d -p 80:80 nginx # 后台运行并映射端口
docker ps                    # 运行中容器
docker ps -a                 # 所有容器
docker stop/start <id>       # 停止/启动
docker rm <id>               # 删除容器
docker logs -f <id>          # 日志
docker exec -it <id> bash    # 进入容器

# 数据
docker volume ls             # 列出卷
docker network ls            # 列出网络

# 清理
docker system prune          # 清理无用资源
docker rmi $(docker images -q) # 删除所有镜像
```

* 一句话记：镜像images/pull/rmi/build，容器run/ps/stop/rm/logs/exec，清理用system prune。