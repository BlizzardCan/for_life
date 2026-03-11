# 一些问题



1. 什么时候应该注释component



2. MQ



3. Redis crud



4. 延时任务 到底用什么去实现比较好



5. 整个业务逻辑搞清楚



6. mysql nosql区别

关联性 结构化 ACID 关联



7. redisd的java客户端



7. 为什么不建议在生产环境使用KEYS





8. 连接配置是写在application.yaml里面(redis、、、、)



9. 为什么redis存入的明明是jack 但看到的却是“乱码” （序列化 -> 可读性差、内存占用较大）

为了更直观 key用Sprin...Serializer 值用jackson2（序列化器）



10. 为了在反序列化时知道对象的类型，json序列化器会将类的class类型写入json结果中，存入redis，会带来额外的内存开销

但为了节省空间，要求需要存储java对象的时候手动完成序列化和反序列化



11. 不是component的话不能用@Autowired，得用构造器？



12. transaction



13. 缓存穿透、缓存雪崩、击穿



14. session tocken jwt



15. lua脚本



16. 可重入锁



17. 秒杀优化 阻塞队列 

后面看下能不能优化为mq实现呢？



18. tablefield



19. 滚动查询

max min offset count



20, geo滚动查询 from？



21. 签到 bit形式

Bitfield key get u[dayOfMonth] 0



22. AOF RDB

持久化



23. 黑马点评 Zset跳表 

（为什么选跳表而不是选择红黑树）



24. RDB

和bgsave



25. kafka是啥

  

26. 这个项目有真实上线运行吗



27. redis主从的数据同步过程

slave什么时候做什么同步



28. 哨兵 心跳

sentinel作用 如何判断节点健康

怎么选的新master 如何进行故障转移的



29. 怎么选举的Leader



30. redistemplate哨兵模式





31. 这个搭建分片集群和哨兵感觉比较冲突啊 分片集群可以互相ping来确定啊



32. 分布式缓存 与 进程本地缓存



33. caffeine



34. 

