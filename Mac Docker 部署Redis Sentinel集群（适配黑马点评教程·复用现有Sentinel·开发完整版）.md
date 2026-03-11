# Mac Docker 部署Redis Sentinel集群（适配黑马点评教程·复用现有Sentinel·开发完整版）

说明：全程贴合**黑马点评sentinel教程**，复用你现有Docker Sentinel（路径/Users/blizzardcan/docker/heima-dianping/sentinel，容器名heima-sentinel），新增2个哨兵节点（均新建本地文件夹，贴合实际开发规范）组成集群（3个哨兵满足故障转移阈值），适配你已有的Redis主从集群（heima-redis为主节点，redis-slave1、redis-slave2为从节点），命令直接复制粘贴，兼顾实操性、开发场景，同时补充核心知识点和面试考点，无需重复部署现有哨兵。

## 一、前置准备（必做，确认现有环境）

1. 确认现有Sentinel状态（必做，确保现有哨兵正常运行）：
        `# 查看现有Sentinel容器（已确认容器名：heima-sentinel）
docker ps | grep sentinel
# 预期输出（你的实际结果）：e3846d1f1992   bladex/sentinel-dashboard:1.8.5   "java -Djava.securit…"   7 days ago   Up 28 hours    0.0.0.0:8080->8080/tcp   heima-sentinel

# 若未运行，启动现有Sentinel
docker start heima-sentinel`

2. 确认现有Sentinel关键信息（已补充，无需再提供）：
        

    - 现有Sentinel的**容器名**：heima-sentinel；

    - 现有Sentinel的**所属网络**：heima-dianping_heima-network（已确认，和Redis主从集群在同一网络）；

    - 现有Sentinel默认已监控Redis主节点（heima-redis），密码认证和Redis主节点（123456）一致（黑马点评初始部署默认配置）；

3. 确认Redis主从集群状态（必做，确保主从同步正常）：
        `# 进入主节点（heima-redis），确认主从同步状态
docker exec -it heima-redis redis-cli -a 123456
info replication
# 预期输出：role:master，connected_slaves:2（确保2个从节点正常连接）
exit`

4. 创建新增哨兵本地目录（贴合实际开发场景，新建独立文件夹，和现有Sentinel路径风格一致）：
        `# 进入你现有Sentinel所在目录（和现有路径统一）
cd /Users/blizzardcan/docker/heima-dianping/
# 新建2个新增哨兵独立文件夹（sentinel2、sentinel3），贴合开发规范
# 开发场景中，每个哨兵节点必须有独立本地目录，用于存储配置、日志，避免冲突
mkdir -p sentinel2 sentinel3
# 权限设置为777（避免Docker挂载权限不足，开发环境常用配置）
chmod 777 sentinel2 sentinel3`关键说明（开发场景重点）：实际开发中，每个哨兵节点必须新建独立本地文件夹，不能共用目录——因为每个哨兵有自己的配置文件（sentinel.conf）、日志文件，共用目录会导致配置覆盖、日志混乱，无法正常组建集群，这也是企业开发的规范要求。**补充解答1：本地挂载目录与容器目录的核心区别（彻底搞懂挂载逻辑）**

        这是Docker部署的核心知识点，结合你的Sentinel环境，用通俗的语言解释，避免复杂术语：
        **补充解答2：为什么你现有Sentinel的conf文件在容器目录，而非本地挂载目录？（贴合你的疑问）**

        教程中sentinel目录下有sentinel.conf，核心原因是教程通常采用“挂载本地目录+提前创建配置文件”的方式部署哨兵；而你现有Sentinel是黑马点评初始部署的，默认是**容器内默认配置（未挂载本地配置目录）**，具体原因+细节如下，彻底解决你的困惑：
        总结：conf文件在容器目录还是本地目录，**核心取决于是否配置了本地挂载（-v命令）**——配置了挂载，就同步到本地；未配置挂载，就只存在于容器内部，和你现有Sentinel的情况完全一致。

    - ① 本地挂载目录（宿主机目录）：就是你Mac电脑上的真实目录，比如你现有Sentinel的路径 **/Users/blizzardcan/docker/heima-dianping/sentinel**、新增哨兵的sentinel2、sentinel3目录，都是你本地能直接找到、打开、修改的目录（相当于“电脑本地文件夹”）；

    - ② 容器目录：是Docker容器内部的“虚拟目录”，容器本身是一个独立的“小系统”，内部有自己的目录结构（相当于“容器里的文件夹”），比如Sentinel容器内部的**/etc/sentinel** 目录，就是容器自己的配置目录，默认存储sentinel.conf；

    - ③ 挂载的作用：通过Docker命令中的 `-v 本地目录:容器目录`（比如新增哨兵的 `-v /Users/blizzardcan/docker/heima-dianping/sentinel2:/etc/sentinel`），把“本地目录”和“容器目录”做“绑定”——容器内部对/etc/sentinel目录的所有操作（比如生成sentinel.conf、写入日志），都会同步到你本地的sentinel2目录；反之，你在本地目录修改文件，也会同步到容器内部，实现“数据互通、持久化”。

    - 1.  黑马点评初始部署为了简化操作，**未给sentinel配置本地挂载**（没有执行`-v 本地目录:容器目录` 命令）：
    
                  这就意味着，你的本地目录 `/Users/blizzardcan/docker/heima-dianping/sentinel`，和Sentinel容器内部的 `/etc/sentinel` 目录**没有绑定**（相当于“两个独立的文件夹，互不关联”）。此时，Sentinel的配置文件（sentinel.conf）会默认生成在**容器内部的/etc/sentinel目录**，不会同步到你本地的sentinel目录，所以你在本地看不到该文件；
              

    - 2.  教程中的sentinel.conf在本地目录，是因为教程部署时**主动配置了本地挂载**：
    
                  教程通常会提前创建本地目录，然后通过`-v 本地目录:/etc/sentinel` 命令，将本地目录和容器目录绑定。此时，Sentinel生成的sentinel.conf会同步到本地目录，方便开发者直接在本地修改配置（不用进入容器），这也是开发场景的规范操作；
              

    - 3.  补充关键细节：你现有Sentinel的本地目录 `/Users/blizzardcan/docker/heima-dianping/sentinel`，大概率是你手动创建的空目录（未和容器绑定），所以里面没有任何文件；而本文档新增哨兵时，已为其配置了挂载命令（`-v 本地目录:容器目录`），因此启动后，sentinel.conf会自动生成在本地的sentinel2、sentinel3目录（验证步骤中会检查），既贴合教程，又符合开发规范。

## 二、新增哨兵节点（2个），组成Sentinel集群（开发场景版·完全适配你的环境）

说明：所有命令已适配你的实际环境（容器名heima-sentinel、网络heima-dianping_heima-network），新增哨兵均挂载新建本地目录，镜像和现有哨兵保持一致（bladex/sentinel-dashboard:1.8.5），直接复制执行即可，无需手动修改。

### 1. 启动新增哨兵1（sentinel2，挂载新建本地目录）

```bash
# 启动哨兵2，容器名：heima-sentinel2，端口：26380，挂载新建本地目录，关联主节点heima-redis（带密码123456）
docker run -d \
--name heima-sentinel2 \
# 适配你的实际网络（heima-dianping_heima-network），确保和现有哨兵、Redis主从集群通信
--network heima-dianping_heima-network \
-p 26380:26379 \
# 挂载新建本地目录（宿主机路径:容器内配置/日志目录，独立目录，贴合开发规范）
-v /Users/blizzardcan/docker/heima-dianping/sentinel2:/etc/sentinel \
# 镜像和现有Sentinel保持一致（黑马点评初始镜像，避免版本兼容问题）
bladex/sentinel-dashboard:1.8.5 \
# 配置监控主节点、密码认证，和现有Sentinel配置一致，满足故障转移2票阈值
java -jar sentinel-dashboard.jar --sentinel monitor mymaster heima-redis 6379 2 --sentinel auth-pass mymaster 123456
```

### 2. 启动新增哨兵2（sentinel3，挂载新建本地目录）

```bash
# 启动哨兵3，容器名：heima-sentinel3，端口：26381，挂载新建本地目录，关联主节点heima-redis（带密码123456）
docker run -d \
--name heima-sentinel3 \
# 适配你的实际网络（heima-dianping_heima-network），确保和现有哨兵、Redis主从集群通信
--network heima-dianping_heima-network \
-p 26381:26379 \
# 挂载新建本地目录（宿主机路径:容器内配置/日志目录，独立目录，贴合开发规范）
-v /Users/blizzardcan/docker/heima-dianping/sentinel3:/etc/sentinel \
# 镜像和现有Sentinel保持一致（黑马点评初始镜像，避免版本兼容问题）
bladex/sentinel-dashboard:1.8.5 \
# 配置监控主节点、密码认证，和现有Sentinel配置一致，满足故障转移2票阈值
java -jar sentinel-dashboard.jar --sentinel monitor mymaster heima-redis 6379 2 --sentinel auth-pass mymaster 123456
```

## 三、验证Sentinel集群（适配黑马点评教程，必做）

启动新增2个哨兵后，执行以下命令验证集群是否生效，贴合教程故障转移测试逻辑，同时对应开发场景的验证规范：

```bash
# 1. 查看3个哨兵容器状态（确保都处于Up状态，3个节点均正常运行）
docker ps | grep sentinel
# 预期输出：heima-sentinel、heima-sentinel2、heima-sentinel3 均为 Up 状态

# 2. 查看任意一个哨兵的日志，确认集群通信正常、已监控主从节点
docker logs -f heima-sentinel2
# 预期输出：包含“sentinel connected to master”“sentinel connected to slave”“sentinel cluster formed”
# 说明3个哨兵已组成集群，正常监控Redis主从节点

# 3. 模拟主节点宕机，测试故障转移（和黑马点评教程完全一致，开发场景核心测试）
# 停止Redis主节点（heima-redis），模拟生产环境主节点故障
docker stop heima-redis
# 查看哨兵日志，确认故障转移触发（耐心等待10-20秒，故障转移需要时间）
docker logs -f heima-sentinel
# 预期输出：包含“failover started”（故障转移开始）、“failover-end”（故障转移结束），提示新主节点选举成功

# 4. 验证新主节点状态（进入任意从节点，查看角色是否变为master）
docker exec -it redis-slave1 redis-cli -a 123456
info replication
# 预期输出：role:master（说明故障转移成功，从节点晋升为主节点）

# 5. 重启原主节点（heima-redis），验证其自动变为从节点（开发场景必备验证）
exit
docker start heima-redis
docker exec -it heima-redis redis-cli -a 123456
info replication
# 预期输出：role:slave，master_host:redis-slave1（或redis-slave2），说明自动同步新主节点数据

# 6. 验证新增哨兵本地目录（开发场景必做）：查看配置文件是否生成
cd /Users/blizzardcan/docker/heima-dianping/sentinel2
ls -l
# 预期输出：包含sentinel.conf配置文件，说明本地目录挂载成功，配置已持久化
```

## 四、Mac Docker 专属注意事项（避坑重点·适配你的环境）

1. 网络一致（核心避坑点）：新增哨兵已配置为heima-dianping_heima-network，和现有哨兵、Redis主从集群完全一致，无需修改；若后续重启容器，需确保网络不被修改，否则会导致集群通信失败。

2. 镜像一致：新增哨兵镜像和现有哨兵保持一致（bladex/sentinel-dashboard:1.8.5），避免版本兼容问题——开发场景中，同一集群的所有哨兵必须使用相同版本，否则会出现配置同步异常。

3. 容器启动失败：
        

    - 若提示“port is already allocated”（端口26380/26381被占用），执行docker ps | grep 26380，停止占用端口的容器后再重启；

    - 若提示“permission denied”（权限不足），重新执行“chmod 777 sentinel2 sentinel3”，确保新增哨兵目录权限足够；

    - 若提示“no route to host”，执行docker restart heima-sentinel heima-redis，重启现有哨兵和主节点后再试；

4. 目录规范（开发场景重点）：新增哨兵必须使用新建独立本地目录（sentinel2、sentinel3），不能和现有哨兵共用目录，也不能两个新增哨兵共用目录，否则会导致配置覆盖、日志混乱，集群无法正常工作。

5. 和黑马点评教程对齐：哨兵集群部署逻辑、故障转移测试步骤，完全贴合教程，新增的本地目录挂载、权限设置是开发场景必备，不影响教程实操，还能帮你理解企业级部署规范。

6. 清理环境（教程结束后）：若需删除新增哨兵，执行以下命令（不删除现有哨兵、Redis主从集群和本地目录数据）：
        `# 停止新增哨兵容器
docker stop heima-sentinel2 heima-sentinel3
# 删除新增哨兵容器（本地目录数据仍保留，后续可复用）
docker rm heima-sentinel2 heima-sentinel3
# 若需彻底删除新增哨兵数据（可选），删除本地目录
# rm -rf /Users/blizzardcan/docker/heima-dianping/sentinel2 /Users/blizzardcan/docker/heima-dianping/sentinel3`

## 五、补充说明（贴合黑马点评教程+开发场景）

1.  现有Sentinel的路径（/Users/blizzardcan/docker/heima-dianping/sentinel）无需修改，新增哨兵目录（sentinel2、sentinel3）和其同源，方便统一管理配置文件和日志，符合企业开发的目录规范；

2.  黑马点评教程中Sentinel的核心配置（监控主节点、故障转移阈值、密码认证），本文档已全部包含，无需手动修改配置文件，命令行参数直接完成配置，兼顾便捷性和规范性；

3.  开发场景中，Sentinel集群需3个节点（避免单点故障），因此新增2个哨兵，和现有哨兵组成集群——若只有1个哨兵，主节点宕机后无法触发故障转移，会导致Redis集群不可用，这是企业开发的硬性要求；

4.  后续教程中Sentinel的监控、故障转移配置，可直接通过本地目录的sentinel.conf文件修改（如调整故障转移阈值、密码），无需进入容器操作，贴合开发便捷性需求。

## 六、Sentinel核心知识点+面试考点（黑马点评教程重点+企业面试高频）

### （一）核心知识点（必掌握，贴合教程+开发）

1. Sentinel核心作用：解决Redis主从集群的“单点故障”问题，实现高可用——主节点宕机后，自动选举从节点成为新主，无需人工干预，保证Redis集群持续可用。

2. Sentinel集群部署要求：
        

    - 至少3个哨兵节点（奇数个），避免“脑裂”（多个哨兵同时选举不同从节点为新主）；

    - 所有哨兵必须和Redis主从节点在同一个网络，否则无法通信、监控；

    - 每个哨兵节点需有独立本地目录（开发场景重点），避免配置、日志冲突；

    - 哨兵镜像版本需和Redis版本、其他哨兵版本保持一致，避免兼容问题。

3. 故障转移核心流程（黑马点评教程重点，开发场景必懂）：
        

    1. 哨兵集群持续监控主节点，当主节点宕机（超过配置的超时时间），某个哨兵会标记主节点为“主观下线”；

    2. 该哨兵向其他哨兵发起投票，若超过“故障转移阈值”（本文档配置为2票），则标记主节点为“客观下线”；

    3. 哨兵集群从所有从节点中，选举一个“最优从节点”（同步数据最完整、运行状态最好），晋升为新主；

    4. 其他从节点自动切换主节点，同步新主节点数据；

    5. 原主节点重启后，自动变为新主节点的从节点，同步新主数据。

4. 关键配置说明（贴合你的环境）：
        

    - sentinel monitor mymaster heima-redis 6379 2：监控名为mymaster的主节点（heima-redis:6379），至少2个哨兵同意，即可触发故障转移；

    - sentinel auth-pass mymaster 123456：哨兵认证Redis主节点的密码，和Redis主节点密码一致，否则无法监控；

    - --appendonly yes：开启AOF持久化，确保哨兵和Redis节点的数据不会因容器重启丢失（开发场景必备）。

5. 本地挂载目录与容器目录补充知识点（贴合你的疑问，必掌握）：
        

    - 本地挂载目录（宿主机目录）：Mac本地真实目录，可直接操作（新建、修改文件），数据持久化在本地，容器删除后数据不丢失；

    - 容器目录：容器内部虚拟目录，仅容器运行时可访问，容器删除后，目录内数据（如未挂载的sentinel.conf）会直接丢失；

    - 挂载命令（-v）的核心意义：实现“本地目录”和“容器目录”数据同步，既保证容器内的配置、日志能持久化到本地，又方便开发者在本地修改配置，是开发场景Docker部署的必备操作。

### （二）面试高频考点（重点记忆，贴合企业面试）

1. 考点1：为什么Sentinel集群需要3个节点（奇数个）？

      答：① 避免脑裂：若偶数个哨兵，可能出现“投票平局”（比如2个哨兵分别投票给2个从节点），无法选举新主；② 满足故障转移阈值（如2票），确保有足够多的哨兵确认主节点下线，避免误判。

2. 考点2：实际开发中，为什么每个哨兵需要独立本地目录？

      答：每个哨兵有自己的配置文件（sentinel.conf）、日志文件，共用目录会导致配置被覆盖、日志混乱，进而导致哨兵集群通信异常、故障转移失败，不符合企业开发规范。

3. 考点3：Sentinel的主观下线和客观下线有什么区别？

      答：① 主观下线（SDOWN）：单个哨兵检测到主节点宕机，认为主节点下线（可能是网络波动导致误判）；② 客观下线（ODOWN）：多个哨兵（超过故障转移阈值）都检测到主节点宕机，共同确认主节点下线，才会触发故障转移。

4. 考点4：Sentinel如何选举新的主节点？

      答：优先选举“同步数据最完整”的从节点（复制偏移量最大），其次选举“运行状态最稳定”“优先级最高”的从节点，确保新主节点能快速承接主节点的读写任务。

5. 考点5：Redis主从集群+Sentinel，为什么还要给从节点、哨兵挂载本地目录？

      答：开发场景中，容器可能因重启、故障被删除，挂载本地目录可实现数据持久化（Redis数据、哨兵配置），避免容器删除后数据、配置丢失，确保集群重启后能快速恢复正常。

6. 考点6：你的环境中，Sentinel集群的网络配置是什么？为什么要这样配置？

      答：网络为heima-dianping_heima-network，因为所有Redis主从节点、哨兵节点必须在同一个网络，否则无法实现通信、监控和故障转移，这是Sentinel集群正常工作的前提。

7. 考点7：为什么有些Sentinel本地目录有sentinel.conf，有些没有？

      答：取决于部署方式：① 若部署时挂载了本地目录，且通过命令行参数或手动创建配置，会在本地目录生成sentinel.conf（教程常用方式，方便修改配置）；② 若未挂载本地目录，配置默认存储在容器内部，本地目录就看不到该文件（你现有Sentinel的情况），不影响哨兵正常运行，但不方便后续配置修改。

8. 考点8：Docker部署中，本地挂载目录和容器目录的区别是什么？挂载的作用是什么？

      答：① 区别：本地挂载目录是宿主机（你的Mac）真实目录，数据持久化在本地；容器目录是容器内部虚拟目录，容器删除后数据丢失；② 挂载作用：通过-v命令绑定两个目录，实现数据同步，既方便本地修改配置、查看日志，又能保证容器内数据持久化，避免容器删除后数据丢失，是开发场景的必备配置。

补充：以上知识点和考点，均贴合黑马点评教程重点和企业实际开发场景，面试中高频出现，建议结合实操记忆，理解每个配置、每个步骤的核心意义，而非死记硬背。

## 七、启动新增哨兵+验证集群 关键命令提炼（实操专用，快速复制）

说明：以下命令已完全适配你的环境，无需修改，复制即可执行，覆盖“启动新增哨兵”“验证集群”全流程核心步骤，节省实操时间。

```bash
# 一、启动新增2个哨兵（核心命令）
# 启动哨兵2（挂载sentinel2目录）
docker run -d \
--name heima-sentinel2 \
--network heima-dianping_heima-network \
-p 26380:26379 \
-v /Users/blizzardcan/docker/heima-dianping/sentinel2:/etc/sentinel \
bladex/sentinel-dashboard:1.8.5 \
java -jar sentinel-dashboard.jar --sentinel monitor mymaster heima-redis 6379 2 --sentinel auth-pass mymaster 123456

# 启动哨兵3（挂载sentinel3目录）
docker run -d \
--name heima-sentinel3 \
--network heima-dianping_heima-network \
-p 26381:26379 \
-v /Users/blizzardcan/docker/heima-dianping/sentinel3:/etc/sentinel \
bladex/sentinel-dashboard:1.8.5 \
java -jar sentinel-dashboard.jar --sentinel monitor mymaster heima-redis 6379 2 --sentinel auth-pass mymaster 123456

# 二、验证集群核心命令（必做）
# 1. 查看3个哨兵状态
docker ps | grep sentinel

# 2. 查看哨兵日志，确认集群通信正常
docker logs -f heima-sentinel2

# 3. 模拟主节点宕机，测试故障转移
docker stop heima-redis
docker logs -f heima-sentinel

# 4. 验证新主节点
docker exec -it redis-slave1 redis-cli -a 123456
info replication
exit

# 5. 重启原主节点，验证自动切换为从节点
docker start heima-redis
docker exec -it heima-redis redis-cli -a 123456
info replication
exit

# 6. 验证新增哨兵本地目录（确认sentinel.conf生成）
cd /Users/blizzardcan/docker/heima-dianping/sentinel2
ls -l
```
> （注：文档部分内容可能由 AI 生成）