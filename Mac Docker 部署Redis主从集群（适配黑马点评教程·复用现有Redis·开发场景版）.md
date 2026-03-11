# Mac Docker 部署Redis主从集群（适配黑马点评教程·复用现有Redis·开发场景版）

说明：全程贴合**实际开发场景**，复用你现有Docker Redis（容器名heima-redis）作为主节点，为从节点挂载本地目录（和主节点路径风格一致），新增从节点和哨兵，命令直接复制粘贴，适配Mac Docker环境，重点解决“主从同步”“哨兵故障转移”，兼顾学习实操和实际开发规范，和黑马点评教程完全匹配。

## 一、前置准备（必做，避免踩坑）

1. 打开Mac Docker Desktop，确保Docker处于运行状态（顶部状态栏有Docker图标，显示“Running”）；

2. 调整Docker资源（可选，避免容器卡顿）：
      

3. 打开Docker Desktop → Settings → Resources；

4. 内存（Memory）调整为2G及以上，点击“Apply & Restart”重启Docker；

5. 确认现有主节点状态（必做，确保主节点正常运行）：
        `# 查看现有Redis主节点（heima-redis）状态，确保Up状态
docker ps | grep heima-redis
# 若未运行，启动主节点
docker start heima-redis`

6. 创建从节点本地目录（实际开发必做，用于挂载配置、持久化数据，和主节点路径一致）：
        `# 进入你主节点所在的目录（和主节点路径风格统一）
cd /Users/blizzardcan/docker/heima-dianping/
# 创建2个从节点目录（slave1、slave2），权限设置为777（避免Docker挂载权限不足）
mkdir -p redis-slave1 redis-slave2
chmod 777 redis-slave1 redis-slave2`

## 二、部署Redis主从集群（复用现有主节点，新增2个从节点·开发版）

全程用命令行操作（Mac打开“终端”，复制以下命令，逐行执行即可），为从节点挂载本地目录（持久化数据），贴合实际开发规范，重点关联你现有主节点heima-redis。

**关键说明：实际开发中必须为从节点挂载本地目录的原因**：开发场景中，从节点需要长期稳定运行，数据需持久化到本地（而非容器内部），避免容器重启/删除导致数据丢失；同时挂载目录可方便修改从节点配置、查看持久化文件，和你现有主节点（/Users/blizzardcan/docker/heima-dianping/redis）的部署规范保持一致，符合企业开发习惯。

**补充说明**：从节点本地目录路径（/Users/blizzardcan/docker/heima-dianping/redis-slave1、redis-slave2），和主节点路径同源，方便后续统一管理，无需额外记忆其他路径。

### 1. 查看现有主节点所属网络（关键：确保主从节点能互相通信）

```bash
# 查看heima-redis容器所属网络（复制输出结果中的“Network”字段，后续用）
docker inspect heima-redis | grep Network
# 示例输出（大概率是默认网络bridge，或自定义网络，复制实际结果）
# "Network": "bridge"
```

说明：后续启动从节点、哨兵，需和主节点（heima-redis）在同一个网络，若输出为bridge（默认网络），无需额外创建网络；若为其他自定义网络，后续命令需添加--network 对应网络名。

### 2. 启动从节点1（slave1），关联主节点+挂载本地目录

```bash
# 启动从节点1，容器名：redis-slave1，端口：6380，挂载本地目录，关联主节点heima-redis（带密码123456）
docker run -d \
--name redis-slave1 \
# 若主节点网络不是bridge，替换下面的bridge为实际网络名（如heima-network）
--network bridge \
-p 6380:6379 \
# 挂载本地目录（宿主机路径:容器内路径），和主节点挂载逻辑一致
-v /Users/blizzardcan/docker/heima-dianping/redis-slave1:/data \
redis:7.0 \
redis-server --appendonly yes --slaveof heima-redis 6379 --requirepass 123456 --masterauth 123456
```

说明：-v 后面的路径是“本地目录:容器内/data目录”，本地目录就是我们刚创建的redis-slave1，数据会同步到本地，容器删除后数据仍保留（开发场景核心需求）。

### 3. 启动从节点2（slave2），关联主节点+挂载本地目录

```bash
# 启动从节点2，容器名：redis-slave2，端口：6381，挂载本地目录，关联主节点heima-redis（带密码123456）
docker run -d \
--name redis-slave2 \
# 若主节点网络不是bridge，替换下面的bridge为实际网络名（如heima-network）
--network bridge \
-p 6381:6379 \
# 挂载本地目录（宿主机路径:容器内路径），和主节点挂载逻辑一致
-v /Users/blizzardcan/docker/heima-dianping/redis-slave2:/data \
redis:7.0 \
redis-server --appendonly yes --slaveof heima-redis 6379 --requirepass 123456 --masterauth 123456
```

### 4. 验证主从同步+本地目录挂载（关键，确认部署成功）

```bash
# 1. 进入现有主节点容器（heima-redis），需输入密码
docker exec -it heima-redis redis-cli -a 123456

# 2. 在主节点写入数据（比如写入黑马点评相关的缓存key，和教程一致）
set shop:101 "XX火锅（西湖店）"

# 3. 退出主节点，进入从节点1，输入密码后查看数据是否同步
exit
docker exec -it redis-slave1 redis-cli -a 123456
get shop:101  # 能查到结果，说明主从同步成功

# 4. 验证本地目录挂载（关键，确认数据持久化到本地）
# 进入从节点1本地目录，查看是否生成持久化文件（appendonly.aof）
cd /Users/blizzardcan/docker/heima-dianping/redis-slave1
ls -l  # 能看到appendonly.aof文件，说明挂载成功、数据持久化到本地

# 5. 同理，验证从节点2（可选）
exit
docker exec -it redis-slave2 redis-cli -a 123456
get shop:101
# 验证从节点2本地目录
cd /Users/blizzardcan/docker/heima-dianping/redis-slave2
ls -l
```

注意：如果查不到数据，大概率是网络问题，重新执行“查看主节点网络”命令，确认从节点--network和主节点一致，再重启2个从节点即可；若本地目录没有aof文件，检查目录权限（确保是777），重新启动从节点。

## 三、部署哨兵模式（Sentinel，适配教程故障转移·开发版）

黑马点评教程会讲哨兵模式，用于主节点宕机后自动选举新主，以下命令适配你现有主节点（heima-redis），带密码认证，直接复制执行，贴合开发场景。

### 1. 启动3个哨兵节点（sentinel1、sentinel2、sentinel3）

```bash
# 启动哨兵1（端口26379），关联主节点heima-redis（带密码123456）
docker run -d \
--name redis-sentinel1 \
# 若主节点网络不是bridge，替换下面的bridge为实际网络名
--network bridge \
-p 26379:26379 \
redis:7.0 \
redis-sentinel --sentinel monitor mymaster heima-redis 6379 2 --sentinel auth-pass mymaster 123456

# 启动哨兵2（端口26380），关联主节点heima-redis（带密码123456）
docker run -d \
--name redis-sentinel2 \
# 若主节点网络不是bridge，替换下面的bridge为实际网络名
--network bridge \
-p 26380:26379 \
redis:7.0 \
redis-sentinel --sentinel monitor mymaster heima-redis 6379 2 --sentinel auth-pass mymaster 123456

# 启动哨兵3（端口26381），关联主节点heima-redis（带密码123456）
docker run -d \
--name redis-sentinel3 \
# 若主节点网络不是bridge，替换下面的bridge为实际网络名
--network bridge \
-p 26381:26379 \
redis:7.0 \
redis-sentinel --sentinel monitor mymaster heima-redis 6379 2 --sentinel auth-pass mymaster 123456
```

说明：哨兵无需挂载本地目录（开发场景中，哨兵日志、配置可无需持久化，容器重启后重新初始化即可，不影响核心功能）；--sentinel auth-pass mymaster 123456 是关键（哨兵认证主节点密码），和教程逻辑一致。

### 2. 验证哨兵功能（模拟主节点宕机，测试故障转移）

```bash
# 1. 停止现有主节点（heima-redis，模拟宕机）
docker stop heima-redis

# 2. 查看任意一个哨兵的日志，确认故障转移（耐心等待10-20秒）
docker logs -f redis-sentinel1

# 3. 故障转移完成后，查看新主节点（一般是slave1或slave2）
# 进入其中一个从节点，输入密码后，执行info replication，查看role是否变为master
docker exec -it redis-slave1 redis-cli -a 123456
info replication

# 4. 测试完成后，可重启原主节点（heima-redis），它会自动变为从节点
exit
docker start heima-redis

# 5. 验证本地数据（开发场景必做）：重启从节点后，查看本地目录aof文件，数据仍保留
cd /Users/blizzardcan/docker/heima-dianping/redis-slave1
cat appendonly.aof  # 能看到之前写入的shop:101数据，说明本地持久化生效
```

注意：故障转移需要10-20秒，日志中会显示“failover-end”，表示转移完成，和黑马点评教程的演示效果一致；重启原主节点后，无需手动配置，会自动同步新主节点数据，且本地目录的数据不会丢失。

## 四、Mac Docker 专属注意事项（避坑重点·开发场景版）

1. 网络匹配：确保从节点、哨兵和主节点（heima-redis）在同一个网络，若之前查看主节点网络不是bridge，务必替换命令中的--network bridge为实际网络名；

2. 容器启动失败：
        

    - 若提示“port is already allocated”（端口被占用），关闭本地已启动的Redis服务（Mac终端执行：brew services stop redis），再重新执行启动命令；

    - 若提示“no route to host”，检查主节点网络是否正确，重启主节点（docker restart heima-redis）后再试；

    - 若提示“permission denied”（权限不足），重新执行“chmod 777 redis-slave1 redis-slave2”，确保本地目录权限足够；

3. 和黑马点评教程对齐：你现有Redis是7.0版本，和教程5.0版本操作逻辑完全一致，仅新增本地目录挂载，贴合开发场景，不影响教程实操；

4. 清理环境（教程结束后）：如果不想保留从节点和哨兵，执行以下命令一键清理（**不删除你的主节点heima-redis，不删除从节点本地目录数据**）：
        `# 停止从节点和哨兵容器
docker stop redis-slave1 redis-slave2 redis-sentinel1 redis-sentinel2 redis-sentinel3

# 删除从节点和哨兵容器（本地目录数据仍保留，后续可复用）
docker rm redis-slave1 redis-slave2 redis-sentinel1 redis-sentinel2 redis-sentinel3

# 若需彻底删除从节点数据（可选），删除本地目录
# rm -rf /Users/blizzardcan/docker/heima-dianping/redis-slave1 /Users/blizzardcan/docker/heima-dianping/redis-slave2

# 若创建了自定义网络，删除网络（未创建则无需执行）
# docker network rm redis-cluster`

## 五、贴合黑马点评教程+开发场景的补充说明

1.  教程中如果涉及“Redis配置文件挂载”，可将配置文件放入从节点本地目录（redis-slave1、redis-slave2），在启动命令中新增配置文件挂载（如 -v 本地目录/redis.conf:/etc/redis/redis.conf），本文档已简化为命令行参数，效果一致，兼顾开发规范和实操便捷；

2.  后续教程中“读写分离”实操（比如黑马点评项目中读请求走从节点），只需在项目配置中，将Redis读地址改为从节点端口（6380或6381），写地址保持主节点（6379），密码填写123456即可，从节点本地数据持久化不影响项目对接；

3.  实际开发中，从节点本地目录会定期备份（避免本地磁盘故障），学习阶段可无需备份，重点掌握“目录挂载+数据持久化”的核心逻辑即可；

4.  你的主节点部署在"/Users/blizzardcan/docker/heima-dianping/redis"，从节点目录和主节点同源，方便后续统一管理，不影响原有部署和数据。
> （注：文档部分内容可能由 AI 生成）