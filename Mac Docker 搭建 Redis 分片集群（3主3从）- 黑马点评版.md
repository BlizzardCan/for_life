# Mac Docker 搭建 Redis 分片集群（3主3从）- 黑马点评版

说明：全程贴合**黑马点评教程**，适配 Mac + Docker 环境，完整实现 3主3从 Redis 分片集群搭建，包含节点启动、集群创建、实操验证、项目整合，同时补充核心知识点和高频面试考点，命令可直接复制粘贴，兼顾实操性、开发场景适配性，无需额外修改环境配置。

## 一、核心信息总览（必看，明确集群架构）

### 1. 集群核心架构（3主3从，标准分片集群）

|节点角色|容器名称|对外端口|本地挂载目录|密码|核心职责|
|---|---|---|---|---|---|
|主节点1|redis-node1|7001|/Users/blizzardcan/docker/heima-dianping/redis-cluster/node1|123456|负责插槽 0-5460，处理读写请求|
|主节点2|redis-node2|7002|/Users/blizzardcan/docker/heima-dianping/redis-cluster/node2|123456|负责插槽 5461-10922，处理读写请求|
|主节点3|redis-node3|7003|/Users/blizzardcan/docker/heima-dianping/redis-cluster/node3|123456|负责插槽 10923-16383，处理读写请求|
|从节点1|redis-node4|7004|/Users/blizzardcan/docker/heima-dianping/redis-cluster/node4|123456|备份主节点1数据，故障时晋升主节点|
|从节点2|redis-node5|7005|/Users/blizzardcan/docker/heima-dianping/redis-cluster/node5|123456|备份主节点2数据，故障时晋升主节点|
|从节点3|redis-node6|7006|/Users/blizzardcan/docker/heima-dianping/redis-cluster/node6|123456|备份主节点3数据，故障时晋升主节点|
### 2. 集群核心规则

- 主从分配：严格遵循 1主1从 绑定，每个主节点对应1个从节点，形成高可用单元，避免单点故障；

- 插槽分配：3个主节点均分 16384个插槽（0~16383），这是Redis分片集群的核心底层逻辑；

- 环境复用：使用黑马点评项目现有Docker网络 heima-dianping_heima-network，与项目其他组件互通；

- 密码统一：所有节点密码均为 123456，适配黑马点评原有Redis环境，避免连接报错；

- 持久化配置：开启AOF持久化，确保容器重启后数据不丢失，贴合开发场景需求。

## 二、前置准备（必做，确认环境+创建目录）

1. 确认Docker环境正常（必做）：
        `# 确认Docker已启动，无报错即正常
docker info

# 确认黑马点评项目网络存在（避免通信失败）
docker network ls | grep heima-dianping_heima-network

# 若网络不存在，执行以下命令创建（一般已存在，无需操作）
# docker network create heima-dianping_heima-network`

2. 创建本地挂载目录（持久化数据，开发场景必做）：
        `# 进入黑马点评项目Docker目录（与现有环境路径统一）
cd /Users/blizzardcan/docker/heima-dianping/

# 创建集群专用目录，包含6个节点的独立文件夹（3主3从）
mkdir -p redis-cluster/{node1,node2,node3,node4,node5,node6}

# 授权（Mac + Docker 必做，解决挂载权限不足问题）
chmod 777 redis-cluster/{node1,node2,node3,node4,node5,node6}`**关键说明**：每个节点独立本地目录，用于存储数据、配置文件和日志，避免不同节点配置覆盖、日志混乱，这是企业开发的规范要求。

3. 补充说明（挂载目录核心逻辑）：


    - 本地挂载目录（宿主机）：即Mac本地真实目录（如上述redis-cluster/node1），可直接打开、修改文件；

    - 容器目录：Redis容器内部的虚拟目录（/data），用于存储容器运行时的数据和配置；

    - 挂载作用：通过 `-v 本地目录:容器目录` 绑定，容器内的操作会同步到本地，实现数据持久化，避免容器删除后数据丢失。

## 三、启动6个Redis节点（3主3从，按顺序执行）

说明：所有命令已适配黑马点评环境，直接复制执行即可，启动后会自动配置集群相关参数（开启集群、设置超时时间、开启持久化）。

### 1. 启动3个主节点（核心分片节点）

```bash
# 启动主节点1（redis-node1，端口7001）
docker run -d \
--name redis-node1 \
--network heima-dianping_heima-network \
-p 7001:6379 \
-v /Users/blizzardcan/docker/heima-dianping/redis-cluster/node1:/data \
redis:7.0 \
redis-server \
--cluster-enabled yes \
--cluster-config-file nodes.conf \
--cluster-node-timeout 5000 \
--appendonly yes \
--requirepass 123456 \
--masterauth 123456

# 启动主节点2（redis-node2，端口7002）
docker run -d \
--name redis-node2 \
--network heima-dianping_heima-network \
-p 7002:6379 \
-v /Users/blizzardcan/docker/heima-dianping/redis-cluster/node2:/data \
redis:7.0 \
redis-server \
--cluster-enabled yes \
--cluster-config-file nodes.conf \
--cluster-node-timeout 5000 \
--appendonly yes \
--requirepass 123456 \
--masterauth 123456

# 启动主节点3（redis-node3，端口7003）
docker run -d \
--name redis-node3 \
--network heima-dianping_heima-network \
-p 7003:6379 \
-v /Users/blizzardcan/docker/heima-dianping/redis-cluster/node3:/data \
redis:7.0 \
redis-server \
--cluster-enabled yes \
--cluster-config-file nodes.conf \
--cluster-node-timeout 5000 \
--appendonly yes \
--requirepass 123456 \
--masterauth 123456
```

### 2. 启动3个从节点（高可用备份节点）

```bash
# 启动从节点1（redis-node4，绑定主节点1）
docker run -d \
--name redis-node4 \
--network heima-dianping_heima-network \
-p 7004:6379 \
-v /Users/blizzardcan/docker/heima-dianping/redis-cluster/node4:/data \
redis:7.0 \
redis-server \
--cluster-enabled yes \
--cluster-config-file nodes.conf \
--cluster-node-timeout 5000 \
--appendonly yes \
--requirepass 123456 \
--masterauth 123456

# 启动从节点2（redis-node5，绑定主节点2）
docker run -d \
--name redis-node5 \
--network heima-dianping_heima-network \
-p 7005:6379 \
-v /Users/blizzardcan/docker/heima-dianping/redis-cluster/node5:/data \
redis:7.0 \
redis-server \
--cluster-enabled yes \
--cluster-config-file nodes.conf \
--cluster-node-timeout 5000 \
--appendonly yes \
--requirepass 123456 \
--masterauth 123456

# 启动从节点3（redis-node6，绑定主节点3）
docker run -d \
--name redis-node6 \
--network heima-dianping_heima-network \
-p 7006:6379 \
-v /Users/blizzardcan/docker/heima-dianping/redis-cluster/node6:/data \
redis:7.0 \
redis-server \
--cluster-enabled yes \
--cluster-config-file nodes.conf \
--cluster-node-timeout 5000 \
--appendonly yes \
--requirepass 123456 \
--masterauth 123456
```

### 3. 验证节点启动状态（必做）

```bash
# 查看6个节点容器状态，需全部为 Up 状态
docker ps | grep redis-node
```

**预期输出**：6个容器均显示 Up 状态，端口分别对应 7001-7006，示例如下：

```bash
redis-node6   redis:7.0   "redis-server --clu…"   Up 1 minute   0.0.0.0:7006->6379/tcp   heima-dianping_heima-network
redis-node5   redis:7.0   "redis-server --clu…"   Up 1 minute   0.0.0.0:7005->6379/tcp   heima-dianping_heima-network
redis-node4   redis:7.0   "redis-server --clu…"   Up 1 minute   0.0.0.0:7004->6379/tcp   heima-dianping_heima-network
redis-node3   redis:7.0   "redis-server --clu…"   Up 1 minute   0.0.0.0:7003->6379/tcp   heima-dianping_heima-network
redis-node2   redis:7.0   "redis-server --clu…"   Up 1 minute   0.0.0.0:7002->6379/tcp   heima-dianping_heima-network
redis-node1   redis:7.0   "redis-server --clu…"   Up 1 minute   0.0.0.0:7001->6379/tcp   heima-dianping_heima-network
```

## 四、创建分片集群（绑定主从+分配插槽，核心步骤）

说明：进入任意节点容器，执行集群创建命令，自动完成主从绑定和插槽分配（--cluster-replicas 1 表示每个主节点配1个从节点）。

```bash
# 1. 进入主节点1容器（任意节点均可，此处选常用节点）
docker exec -it redis-node1 bash

# 2. 执行集群创建命令（关键！自动分配主从+插槽）
redis-cli --cluster create \
redis-node1:6379 \
redis-node2:6379 \
redis-node3:6379 \
redis-node4:6379 \
redis-node5:6379 \
redis-node6:6379 \
--cluster-replicas 1 \
-a 123456
```

**必做操作**：执行后终端会提示 `Can I set the above configuration? (type 'yes' to accept):`，输入 **yes** 并回车，等待集群创建完成（提示`[OK] All nodes created.` 即成功）。

### 验证集群状态（必做，确认主从+插槽分配）

```bash
# 1. 以集群模式连接主节点1（-c 必须加，否则无法自动路由）
redis-cli -c -p 7001 -a 123456

# 2. 查看集群节点核心信息（主从角色、插槽分配）
cluster nodes
```

**预期结果（重点对照）**：

- redis-node1（主节点）：role:master，负责插槽 0-5460；

- redis-node2（主节点）：role:master，负责插槽 5461-10922；

- redis-node3（主节点）：role:master，负责插槽 10923-16383；

- redis-node4（从节点）：role:slave，master 字段为 redis-node1 的 ID；

- redis-node5（从节点）：role:slave，master 字段为 redis-node2 的 ID；

- redis-node6（从节点）：role:slave，master 字段为 redis-node3 的 ID。

## 五、实操验证（分片+故障转移，贴合教程+开发）

### 1. 测试分片集群（自动路由，验证插槽分配）

```bash
# 保持集群连接状态（若断开，重新执行：redis-cli -c -p 7001 -a 123456）

# 写入测试数据（不同key会自动路由到对应主节点）
set shop:101 "黑马点评-西湖火锅店"
set user:10086 "用户10086（黑马点评）"
set coupon:2024 "满减券-2024（黑马点评）"
set id:999 "商品ID-999（黑马点评）"

# 读取数据（自动跳转到对应主节点）
get shop:101
get user:10086
get coupon:2024
get id:999
```

**操作现象**：执行 set/get 命令时，终端会显示 `-> Redirected to slot [xxx] node xxx:6379`，说明分片路由正常。

### 2. 测试故障转移（模拟主节点宕机，验证高可用）

```bash
# 1. 模拟主节点1宕机（停止redis-node1容器）
docker stop redis-node1

# 2. 查看集群故障转移状态（等待5-10秒，集群自动选举）
redis-cli -c -p 7002 -a 123456 cluster nodes

# 3. 恢复原主节点1（验证自动归为从节点）
docker start redis-node1

# 4. 再次查看节点状态，确认主从关系
redis-cli -c -p 7002 -a 123456 cluster nodes
```

**预期结果**：

- 主节点1宕机后：redis-node1 状态变为 fail，redis-node4（原从节点）晋升为新主节点，接管插槽 0-5460；

- 主节点1恢复后：自动变为 redis-node4 的从节点，同步新主节点数据。

## 六、黑马点评项目整合（RedisTemplate 访问分片集群）

### 1. 修改项目配置文件（application.yml）

替换原有单机Redis配置，开启集群模式，适配分片集群：

```yaml
spring:
  redis:
    # 集群统一密码，与节点一致
    password: 123456
    # 分片集群核心配置：列出所有节点地址
    cluster:
      nodes:
        - 127.0.0.1:7001
        - 127.0.0.1:7002
        - 127.0.0.1:7003
        - 127.0.0.1:7004
        - 127.0.0.1:7005
        - 127.0.0.1:7006
      # 最大重定向次数（固定值，无需修改）
      max-redirects: 3
    # 连接池配置（复用黑马点评原有配置）
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
        max-wait: -1ms
```

### 2. 代码使用（与单机Redis完全一致）

项目代码无需任何修改，StringRedisTemplate 会自动处理分片路由逻辑：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class RedisClusterController {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    // 测试分片集群读写（和单机Redis用法完全相同）
    @GetMapping("/test/cluster")
    public String testCluster() {
        // 1. 写入数据（自动分片到对应节点）
        stringRedisTemplate.opsForValue().set("shop:101", "黑马点评-西湖火锅店");
        // 2. 读取数据（自动路由到存储节点）
        String value = stringRedisTemplate.opsForValue().get("shop:101");
        // 3. 返回结果
        return "分片集群测试成功！key=shop:101，value=" + value;
    }
}
```

## 七、Mac Docker 专属注意事项（避坑重点）

1. 网络一致：所有节点均配置为 heima-dianping_heima-network，与黑马点评项目其他组件互通，重启容器后需确保网络不变；

2. 镜像版本：所有Redis节点均使用 redis:7.0，避免版本兼容问题，开发场景中同一集群节点版本必须一致；

3. 容器启动失败解决方案：
        

    - 端口占用：提示“port is already allocated”，执行 `docker ps | grep 7001`（替换对应端口），停止占用端口的容器后重启；

    - 权限不足：提示“permission denied”，重新执行 `chmod 777 redis-cluster/{node1,node2,node3,node4,node5,node6}`；

    - 通信失败：提示“no route to host”，执行 `docker restart redis-node1 redis-node2 redis-node3`，重启主节点后再试。

4. 目录规范：每个节点必须使用独立本地目录，不可共用，否则会导致配置覆盖、日志混乱，集群无法正常工作；

5. 环境清理（教程结束后可选）：
`# 停止所有集群节点
docker stop redis-node1 redis-node2 redis-node3 redis-node4 redis-node5 redis-node6

# 删除所有集群节点容器
docker rm redis-node1 redis-node2 redis-node3 redis-node4 redis-node5 redis-node6

# 彻底删除本地数据（可选）
rm -rf /Users/blizzardcan/docker/heima-dianping/redis-cluster`

## 八、核心知识点+高频面试考点（黑马点评教程重点）

### （一）核心知识点（必掌握）

1. 3主3从架构核心逻辑：
       

    - 主节点：负责分片存储数据、处理读写请求、参与故障转移投票；

    - 从节点：负责备份主节点数据、故障时晋升为主节点、分担读请求（可选）；

    - 核心：1主1从形成高可用单元，集群整体为多主多从架构，突破单机容量限制。

2. 散列插槽（Hash Slots）核心：
        

    - 总数：16384个（0~16383），3个主节点均分；

    - 算法：`slot = CRC16(key) & 16383`，根据key计算插槽，实现自动路由；

    - 作用：解决数据分片问题，将海量数据分散到多个节点，提升并发能力。

3. 故障转移流程：
        

    1. 主节点宕机超过 cluster-node-timeout（5秒），从节点标记主节点为“主观下线”；

    2. 从节点向其他主节点发起投票，超过半数投票通过，标记主节点为“客观下线”；

    3. 从节点晋升为新主节点，接管原主节点插槽；

    4. 原主节点恢复后，自动变为新主节点的从节点。

### （二）高频面试考点（重点背诵）

1. 考点1：Redis集群的槽位（Hash Slots）是怎么回事？为什么是16384个？
        

    - 定义：Redis集群将数据空间划分为16384个槽位，每个主节点负责一部分，数据写入时通过算法计算槽位，路由到对应节点；

    - 原因：① 心跳包大小限制（2KB可描述所有槽位）；② 哈希冲突概率低，数据分布均匀；③ 平衡性能与分布合理性。

2. 考点2：分片集群和主从集群的区别？
        

    - 主从集群：1主多从，全量复制，解决高可用，无法突破单机容量；

    - 分片集群：多主多从，分片存储，解决容量瓶颈+高可用，提升并发能力。

3. 考点3：RedisTemplate访问分片集群需要注意什么？
        

    - 配置：需配置 cluster.nodes 列出所有节点，而非单机地址；

    - 限制：不支持跨节点事务、跨槽位批量操作（如mget/mset）；

    - 优势：自动路由，故障自动感知，无需修改代码。

## 九、文档使用说明

- 1.  实操时按章节顺序执行命令，重点关注“前置准备”“集群创建”“验证步骤”，避免跳过关键操作；

- 2.  命令可直接复制粘贴到终端执行，无需手动修改路径、密码等信息（已适配你的环境）；

- 3.  面试前重点复习“核心知识点+面试考点”，结合实操记忆，贴合黑马点评教程重点；

- 4.  可直接复制本文档内容，保存为.md文件（适配Typora/VS Code），方便本地离线查看。
> （注：文档部分内容可能由 AI 生成）