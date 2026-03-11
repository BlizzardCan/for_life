# Docker 在黑马点评项目中的高效使用指南

你现在已经把Docker的基础环境（MySQL/Redis/Nacos等）跑起来了，也踩过可视化工具的坑，现在我用**“大白话+实用场景”** 给你捋清楚：Docker接下来该怎么用、能怎么用，聚焦你学习黑马点评项目的核心需求，不搞虚的～

### 核心定位：Docker对你的价值

对学习黑马点评项目来说，Docker就是你的**“本地一站式服务环境”** ——不用手动装MySQL/Redis/Nacos，不用配端口/密码冲突，一键启动/停止所有依赖服务，这是Docker最核心的用法。

## 一、日常使用：高频操作（每天都会用到）

这些命令是你接下来用Docker的“基本功”，记熟这几个就够了：

### 1. 查看所有运行中的服务（确认环境是否正常）

```Bash
# 看正在运行的容器（能看到MySQL/Redis/Nacos等）
docker ps
# 看所有容器（包括停止的）
docker ps -a
```

✅ 每天启动项目前，先执行`docker ps`，确认`heima-mysql`/`heima-redis`/`heima-nacos`等都是`Up`状态。

### 2. 启动/停止/重启服务（按需求操作）

```Bash
# 启动单个服务（比如Redis）
docker start heima-redis
# 停止单个服务（比如MySQL）
docker stop heima-mysql
# 重启所有黑马点评相关服务（推荐用docker-compose）
cd ~/docker/heima-dianping  # 进入你的docker-compose.yml目录
docker compose restart
```

✅ 关机前执行`docker compose stop`，开机后执行`docker compose up -d`，一键启停所有服务，不用逐个启动。

### 3. 查看服务日志（排错必备）

项目启动报错（比如连不上MySQL），先看Docker服务的日志：

```Bash
# 查看MySQL日志（找连接失败/启动报错）
docker logs heima-mysql
# 实时查看Redis日志（看有没有连接请求）
docker logs -f heima-redis
```

### 4. 进入服务内部（调试/操作）

比如想手动操作MySQL、执行Redis命令，不用可视化工具也能行：

```Bash
# 进入MySQL容器，直接操作数据库
docker exec -it heima-mysql mysql -uroot -proot
# 进入Redis容器，执行指令
docker exec -it heima-redis redis-cli -a 123456
# 进入Nacos容器，查看配置
docker exec -it heima-nacos bash
```

## 二、进阶用法：适配黑马点评项目（提升效率）

### 1. 用docker-compose统一管理所有服务（核心推荐）

你之前应该有`docker-compose.yml`文件，这是Docker最核心的高效用法——**一个文件管理所有服务**，不用记一堆`docker run`命令：

```Bash
# 一键启动所有服务（后台运行）
cd ~/docker/heima-dianping
docker compose up -d

# 一键停止所有服务
docker compose stop

# 一键删除所有服务（不用了的时候）
docker compose down
```

✅ 把MySQL/Redis/Nacos/RabbitMQ/Sentinel都写进`docker-compose.yml`，每次启动项目只需要这一条命令，比逐个启动爽10倍。

### 2. 数据持久化（避免重启容器丢数据）

黑马点评的MySQL数据、Redis缓存不能丢，所以你的`docker-compose.yml`里一定要有**挂载目录**（之前应该配过），比如：

```YAML
heima-mysql:
  image: mysql:8.0
  volumes:
    - ~/docker/heima-dianping/mysql/data:/var/lib/mysql  # 数据持久化到本地
    - ~/docker/heima-dianping/mysql/conf:/etc/mysql/conf.d
heima-redis:
  image: redis:7.0
  volumes:
    - ~/docker/heima-dianping/redis/data:/data  # Redis数据持久化
```

✅ 就算删了MySQL容器，`~/docker/heima-dianping/mysql/data`里的数据库数据还在，重新启动容器就能恢复。

### 3. 端口映射（确保项目能访问）

确认`docker-compose.yml`里的端口映射和你项目配置一致：

- MySQL：`3306:3306`
- Redis：`6379:6379`
- Nacos：`8848:8848`
- RabbitMQ：`5672:5672`、`15672:15672`
- Sentinel：`8080:8080`

### 4. 导入黑马点评的SQL脚本（初始化数据库）

如果项目启动提示“表不存在”，把SQL脚本导入MySQL容器：

```Bash
# 假设你的SQL文件在~/Downloads/heima-dianping.sql
docker exec -i heima-mysql mysql -uroot -proot heima < ~/Downloads/heima-dianping.sql
```

## 三、避坑指南：新手最容易踩的坑

1. **端口冲突**：比如本地装了MySQL，占用3306端口，Docker的MySQL就启动不了→改Docker的映射端口（比如`3307:3306`），并修改项目配置里的端口。
2. **容器启动顺序**：Nacos要先启动，再启动项目；MySQL没启动，项目连不上→用`docker compose up -d`，Docker会自动处理依赖。
3. **密码不一致**：项目配置里的Redis密码是`123456`，Docker里的Redis密码是`root`→必须保持一致，否则连接失败。
4. **数据丢失**：没配`volumes`挂载→删容器前先备份`~/docker/heima-dianping`目录。

## 四、Docker能帮你做的更多事（拓展）

除了跑黑马点评的依赖服务，Docker还能：

1. **跑前端项目**：把Vue/React项目打包成Docker镜像，一键启动，不用配Node环境；
2. **模拟生产环境**：在本地模拟多服务部署（比如微服务集群），和生产环境一致；
3. **快速切换版本**：比如想测试MySQL 5.7和8.0的区别，只需要改`docker-compose.yml`里的镜像版本，一键重启。

### 总结（核心用法回顾）

1. **日常操作**：`docker ps`（看状态）、`docker compose up -d`（启动所有服务）、`docker logs`（排错）；
2. **核心价值**：一站式管理黑马点评的所有依赖服务，不用手动装/配任何中间件；
3. **关键配置**：`docker-compose.yml`里配好**端口映射**和**数据挂载**，避免端口冲突、数据丢失。

接下来你只需要：

1. 用`docker compose up -d`启动所有服务；
2. 确认项目配置里的端口/密码和Docker一致；
3. 启动黑马点评项目，测试功能。

有任何Docker相关的报错（比如服务启动失败、项目连不上服务），直接告诉我报错信息，我帮你秒定位～