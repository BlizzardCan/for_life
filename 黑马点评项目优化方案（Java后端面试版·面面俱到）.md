# 黑马点评项目优化方案（Java后端面试版·面面俱到）

# 一、文档前言

本文档针对经典Java后端项目「黑马点评」（本地生活O2O平台）进行增量优化，核心目标是：**不重构原有项目、不新增复杂业务，仅通过技术升级，融入AI、多级缓存、MCP协议、高并发秒杀、流式接口等高频面试知识点**，既保留项目原有业务完整性，又提升技术深度和面试竞争力，适配Java后端实习/校招初级岗面试需求。

优化核心原则：所有新增功能均贴合黑马点评本地生活业务场景（商户查询、优惠券秒杀、用户交互等），不脱离业务空谈技术，确保面试时能清晰讲清「技术如何解决业务问题」，体现Java后端工程能力。

# 二、原有项目基础（铺垫，面试必讲）

## 2.1 原有核心功能

- 商户模块：商户查询、商户详情展示、附近商户检索（基于Redis Geo）

- 优惠券模块：优惠券发放、领取、核销，基础秒杀功能

- 用户模块：用户注册、登录、签到（基于Redis Bitmap）、个人中心

- 互动模块：用户点赞、评论、收藏，博客分享

## 2.2 原有技术栈（基础版）

Java 17、Spring Boot 3.x、Spring MVC、MyBatis-Plus、MySQL、Redis、Git、Maven

## 2.3 原有项目痛点（优化切入点）

- 缓存设计简单：仅用Redis单机缓存，未考虑缓存穿透、雪崩问题，响应速度有优化空间

- 高并发支撑不足：优惠券秒杀用简单Redis锁，集群环境下易失效，无异步处理机制

- 用户交互体验一般：查询商户信息需手动翻阅详情，无智能交互方式

- 技术亮点单一：仅覆盖基础后端技术，缺乏AI、响应式编程等热门加分项

# 三、详细优化功能（核心部分，面面俱到）

本次优化共4个核心模块，按「实现难度从低到高」排序，每个模块包含：业务场景、优化目的、实现步骤、核心代码、技术栈体现、面试亮点，确保你能看懂、会实现、能讲清。

## 优化1：AI智能客服（含RAG检索+MCP协议集成）

### 3.1.1 业务场景

用户查询商户时，无需手动翻阅详情，可通过自然语言向AI客服提问，例如：「这家店的招牌菜是什么？」「满200减50的优惠券怎么用？」「这家店的奶茶还有库存吗？」，AI将结合商户知识库+实时业务数据（通过MCP调用工具）给出精准回答，提升用户交互体验。

### 3.1.2 优化目的

- 融入Spring AI、RAG检索、MCP协议等热门知识点，丰富技术亮点

- 体现Java后端对AI应用的落地能力，区别于普通实习生项目

- 贴合业务场景，不牵强，面试时易引导面试官深挖技术细节

### 3.1.3 实现步骤（详细可落地，1-2小时搞定）

#### 步骤1：引入核心依赖（Spring AI、MCP相关）

```xml
<!-- Spring AI 核心依赖（对接大模型） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-core</artifactId>
    <version>1.0.0-M1</version>
</dependency>
<!-- 阿里云百炼大模型依赖（不用本地部署Ollama，省时间） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-alibaba-tongyi</artifactId>
    <version>1.0.0-M1</version>
</dependency>
<!-- Spring AI MCP协议依赖（工具调用） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp</artifactId>
    <version>1.0.0-M1</version>
</dependency>
<!-- Redis（会话记忆、知识库缓存） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
    
```

#### 步骤2：配置大模型与MCP协议

```java

// 1. 大模型配置（阿里云百炼，不用本地部署，填自己的API Key即可）
@Configuration
public class AiConfig {
    // 从配置文件读取阿里云百炼API Key
    @Value("${spring.ai.alibaba.tongyi.api-key}")
    private String apiKey;

    // 配置ChatClient（大模型调用入口）
    @Bean
    public ChatClient chatClient(McpAgents mcpAgents) {
        return ChatClient.builder()
                .provider(new AlibabaTongyiChatModel(AlibabaTongyiChatOptions.builder()
                        .apiKey(apiKey)
                        .model("qwen-turbo") // 轻量模型，响应快，适合测试
                        .build()))
                .mcpAgents(mcpAgents) // 绑定MCP工具，让大模型能调用Java方法
                .build();
    }
}

// 2. MCP协议配置（注册工具方法）
@Configuration
public class McpConfig {
    // 注入自定义的MCP工具服务（封装黑马点评业务方法）
    @Bean
    public McpAgents mcpAgents(ShopMcpToolService shopMcpToolService) {
        return McpAgents.builder()
                .add(shopMcpToolService) // 注册MCP工具
                .build();
    }
}
    
```

#### 步骤3：封装MCP工具服务（核心，让AI调用Java业务方法）

将黑马点评原有业务方法（查库存、查优惠券数量等）封装成MCP工具，让AI能通过MCP协议自动调用，核心是Spring AI的@McpService、@McpFunction注解。

```java

@Service
@McpService("heima_dianping_mcp_tools") // MCP工具服务名称，给大模型识别
public class ShopMcpToolService {
    // 注入黑马点评原有业务Service
    private final ShopService shopService;
    private final CouponService couponService;
    private final UserService userService;

    // 构造器注入（避免Autowired，面试时更显规范）
    public ShopMcpToolService(ShopService shopService, CouponService couponService, UserService userService) {
        this.shopService = shopService;
        this.couponService = couponService;
        this.userService = userService;
    }

    /**
     * MCP工具方法1：查询商户指定商品的实时库存
     * 注解说明：@McpFunction告诉大模型这是可调用的工具，description是给大模型看的，必须清晰
     */
    @McpFunction(
            name = "get_shop_goods_stock",
            description = "查询指定商户的指定商品实时库存，适用于用户问商品是否有货的场景，参数必填"
    )
    public Integer getShopGoodsStock(
            @McpParameter(name = "shopId", description = "商户ID，唯一标识，必填") Long shopId,
            @McpParameter(name = "goodsId", description = "商品ID，唯一标识，必填") Long goodsId
    ) {
        // 调用黑马点评原有业务方法（核心还是Java后端开发，MCP只是调用协议）
        return shopService.getRealStock(shopId, goodsId);
    }

    /**
     * MCP工具方法2：查询指定优惠券的剩余数量
     */
    @McpFunction(
            name = "get_coupon_remain_count",
            description = "查询指定优惠券的剩余可领取数量，适用于用户问优惠券是否还有的场景，参数必填"
    )
    public Integer getCouponRemainCount(
            @McpParameter(name = "couponId", description = "优惠券ID，唯一标识，必填") Long couponId
    ) {
        return couponService.getRemainCount(couponId);
    }

    /**
     * MCP工具方法3：查询用户当月签到次数
     */
    @McpFunction(
            name = "get_user_check_in_count",
            description = "查询指定用户当月的签到次数，适用于用户问自己签到情况的场景，参数必填"
    )
    public Integer getUserCheckInCount(
            @McpParameter(name = "userId", description = "用户ID，唯一标识，必填") Long userId
    ) {
        return userService.getMonthCheckInCount(userId);
    }
}
    
```

#### 步骤4：实现RAG知识库与会话记忆

RAG核心是「检索增强生成」，将黑马点评的商户详情、优惠券规则等静态数据做成知识库，AI先检索知识库，再结合MCP工具调用的实时数据，给出精准回答；会话记忆用Redis存储，让AI记住用户之前的对话。

```java

@Service
public class ShopAICustomerService {
    private final ChatClient chatClient;
    private final StringRedisTemplate redisTemplate;
    // 知识库缓存key前缀（存商户静态信息：招牌菜、营业时间等）
    private static final String SHOP_KNOWLEDGE_KEY = "shop:knowledge:";
    // 会话记忆key前缀（存用户对话历史）
    private static final String CHAT_HISTORY_KEY = "chat:history:";

    public ShopAICustomerService(ChatClient chatClient, StringRedisTemplate redisTemplate) {
        this.chatClient = chatClient;
        this.redisTemplate = redisTemplate;
    }

    /**
     * AI客服核心方法：结合RAG知识库+MCP工具调用+会话记忆
     * @param shopId 商户ID（关联知识库）
     * @param userQuestion 用户问题
     * @param userId 用户ID（关联会话记忆）
     * @return AI回答
     */
    public String askShopAI(Long shopId, String userQuestion, String userId) {
        // 1. 检索RAG知识库（从Redis获取该商户的静态信息）
        String shopKnowledge = getShopKnowledge(shopId);
        // 2. 获取用户会话记忆（从Redis获取之前的对话）
        String chatHistory = getChatHistory(userId);
        // 3. 拼接Prompt（引导大模型：先查知识库，不够再调用MCP工具）
        String prompt = buildPrompt(shopKnowledge, chatHistory, userQuestion);
        // 4. 调用大模型，自动判断是否需要调用MCP工具（无需手动干预）
        ChatResponse response = chatClient.prompt().user(prompt).call();
        // 5. 保存最新的对话记忆到Redis（更新会话）
        saveChatHistory(userId, userQuestion, response.getContent());
        // 6. 返回AI回答
        return response.getContent();
    }

    // 从Redis获取商户知识库（项目启动时可预热，把数据库商户信息存入Redis）
    private String getShopKnowledge(Long shopId) {
        String knowledge = redisTemplate.opsForValue().get(SHOP_KNOWLEDGE_KEY + shopId);
        // 若未找到，从数据库查询并缓存（兜底）
        if (StrUtil.isBlank(knowledge)) {
            Shop shop = shopService.getById(shopId);
            knowledge = buildShopKnowledge(shop); // 拼接商户静态信息（招牌菜、营业时间等）
            redisTemplate.opsForValue().set(SHOP_KNOWLEDGE_KEY + shopId, knowledge, 7, TimeUnit.DAYS);
        }
        return knowledge;
    }

    // 拼接商户知识库内容（示例）
    private String buildShopKnowledge(Shop shop) {
        return "商户名称：" + shop.getName() + "\n" +
                "商户地址：" + shop.getAddress() + "\n" +
                "营业时间：" + shop.getOpenTime() + "-" + shop.getCloseTime() + "\n" +
                "招牌菜：" + shop.getSignatureDish() + "\n" +
                "人均消费：" + shop.getPerCapita() + "元\n" +
                "优惠活动：" + shop.getPromotion();
    }

    // 获取用户会话记忆（保留最近10条对话，避免上下文过长）
    private String getChatHistory(String userId) {
        String history = redisTemplate.opsForValue().get(CHAT_HISTORY_KEY + userId);
        return StrUtil.isBlank(history) ? "无对话历史" : history;
    }

    // 保存用户会话记忆（拼接新对话，限制长度）
    private void saveChatHistory(String userId, String userQuestion, String aiResponse) {
        String oldHistory = getChatHistory(userId);
        String newHistory = oldHistory + "\n用户：" + userQuestion + "\nAI：" + aiResponse;
        // 截取最近10条对话（简单优化，避免Redis存储过大）
        if (newHistory.split("\n").length > 20) {
            String[] lines = newHistory.split("\n");
            newHistory = String.join("\n", Arrays.copyOfRange(lines, lines.length - 20, lines.length));
        }
        redisTemplate.opsForValue().set(CHAT_HISTORY_KEY + userId, newHistory, 1, TimeUnit.DAYS);
    }

    // 拼接Prompt（核心：引导大模型正确使用知识库和MCP工具）
    private String buildPrompt(String shopKnowledge, String chatHistory, String userQuestion) {
        return "你是黑马点评的AI客服，负责回答用户关于商户的各类问题，规则如下：" +
                "1. 先优先从提供的商户知识库中查找答案，知识库内容：" + shopKnowledge +
                "2. 若知识库中没有相关信息（比如实时库存、优惠券剩余数量），请调用提供的MCP工具获取实时数据，不要凭空猜测" +
                "3. 记住用户的对话历史：" + chatHistory + "，保持对话连贯性" +
                "4. 回答简洁明了，贴合用户问题，不要添加无关内容" +
                "用户当前问题：" + userQuestion;
    }
}
    
```

#### 步骤5：编写AI客服接口（Controller层）

```java

@RestController
@RequestMapping("/api/ai/customer")
public class ShopAIController {
    private final ShopAICustomerService aiCustomerService;

    public ShopAIController(ShopAICustomerService aiCustomerService) {
        this.aiCustomerService = aiCustomerService;
    }

    /**
     * 同步AI客服接口（基础版）
     */
    @GetMapping("/ask")
    public Result askAI(
            @RequestParam Long shopId,
            @RequestParam String userQuestion,
            @RequestParam String userId
    ) {
        String aiResponse = aiCustomerService.askShopAI(shopId, userQuestion, userId);
        return Result.ok(aiResponse);
    }

    // 流式接口将在「优化4」中实现，此处先保留同步接口
}
```

### 3.1.4 技术栈体现（面试重点讲）

- Spring AI：对接大模型（阿里云百炼），实现RAG检索、MCP工具调用，核心是Java生态的AI应用落地

- MCP协议：Model Control Protocol，通过Spring AI注解封装Java业务方法，让AI自动调用外部工具，体现接口封装和协议应用能力

- Redis：存储RAG知识库、用户会话记忆，体现缓存设计能力

- Java基础：面向对象编程、接口封装、异常处理，体现后端开发基本功

### 3.1.5 面试亮点

能讲清「AI客服如何结合业务落地」，区别于纯理论的AI项目；能说明MCP协议的作用（让AI调用Java工具），体现技术广度；能讲清RAG检索和会话记忆的实现逻辑，体现工程落地能力。

## 优化2：多级缓存架构升级（Caffeine+Redis+布隆过滤器）

### 3.2.1 业务场景

黑马点评的「商户详情查询」「首页热点商户列表」是高频读请求（QPS可达数千），原有仅用Redis缓存，存在响应速度不够快、缓存穿透、缓存雪崩等问题。升级为「本地缓存（Caffeine）+ Redis分布式缓存 + 布隆过滤器」的多级架构，提升响应速度，解决缓存相关问题。

### 3.2.2 优化目的

- 覆盖Java后端高频考点：多级缓存、缓存穿透/雪崩/击穿的解决方案

- 体现性能优化能力，面试时能讲清「如何通过缓存优化提升系统性能」

- 贴合高并发场景，符合后端开发的核心需求，提升项目竞争力

### 3.2.3 实现步骤（详细可落地，1小时搞定）

#### 步骤1：引入核心依赖（Caffeine、Redisson）

```xml
<!-- Caffeine本地缓存（高性能本地缓存） -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>
<!-- Redisson（分布式锁、布隆过滤器） -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.23.3</version>
</dependency>
    
```

#### 步骤2：配置Caffeine本地缓存和Redisson布隆过滤器

```java
// 1. Caffeine本地缓存配置
@Configuration
public class CaffeineConfig {
    /**
     * 热点商户本地缓存：存最热的100个商户详情，10分钟过期
     * 配置说明：maximumSize（最大缓存数量）、expireAfterWrite（写入后过期时间）
     */
    @Bean
    public Cache<Long, Shop> shopLocalCache() {
        return Caffeine.newBuilder()
                .maximumSize(100) // 缓存最多100个商户，避免内存溢出
                .expireAfterWrite(10, TimeUnit.MINUTES) // 10分钟过期，保证数据新鲜
                .recordStats() // 记录缓存统计信息（命中率等，可选，面试时可讲）
                .build();
    }
}

// 2. Redisson布隆过滤器配置（防止缓存穿透）
@Configuration
public class RedissonConfig {
    @Bean
    public RedissonClient redissonClient() {
        // 基础配置（本地Redis，根据自己的环境修改）
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        return Redisson.create(config);
    }

    // 布隆过滤器：用于拦截无效商户ID查询，防止缓存穿透
    @Bean
    public RedissonBloomFilter<Long> shopIdBloomFilter(RedissonClient redissonClient) {
        // 获取布隆过滤器（不存在则创建）
        RedissonBloomFilter<Long> bloomFilter = redissonClient.getBloomFilter("shop:id:bloom");
        // 初始化布隆过滤器：预计数据量10000，误判率0.01（行业常用配置）
        bloomFilter.tryInit(10000, 0.01);
        return bloomFilter;
    }
}
    
```

#### 步骤3：布隆过滤器预热（项目启动时加载商户ID）

项目启动时，将数据库中所有商户ID加载到布隆过滤器，确保能拦截无效ID查询，避免缓存穿透。

```java

// 项目启动器，加载布隆过滤器数据
@Component
public class BloomFilterInitializer implements CommandLineRunner {
    private final RedissonBloomFilter<Long> shopIdBloomFilter;
    private final ShopMapper shopMapper;

    public BloomFilterInitializer(RedissonBloomFilter<Long> shopIdBloomFilter, ShopMapper shopMapper) {
        this.shopIdBloomFilter = shopIdBloomFilter;
        this.shopMapper = shopMapper;
    }

    // 项目启动后执行
    @Override
    public void run(String... args) throws Exception {
        // 从数据库查询所有商户ID
        List<Long> shopIds = shopMapper.selectList(null).stream()
                .map(Shop::getId)
                .collect(Collectors.toList());
        // 将所有商户ID添加到布隆过滤器
        for (Long shopId : shopIds) {
            shopIdBloomFilter.add(shopId);
        }
        System.out.println("布隆过滤器预热完成，共加载商户ID：" + shopIds.size() + "个");
    }
}
    
```

#### 步骤4：改造商户查询接口，实现多级缓存流程

核心流程：本地缓存（Caffeine）→ 布隆过滤器 → Redis缓存 → 数据库，同时实现缓存回写、动态TTL防缓存雪崩。

```java

@Service
public class ShopService {
    private final ShopMapper shopMapper;
    private final StringRedisTemplate redisTemplate;
    private final Cache<Long, Shop> shopLocalCache; // Caffeine本地缓存
    private final RedissonBloomFilter<Long> shopIdBloomFilter; // 布隆过滤器
    // Redis缓存key前缀
    private static final String SHOP_REDIS_KEY = "shop:cache:";

    // 构造器注入所有依赖
    public ShopService(ShopMapper shopMapper, StringRedisTemplate redisTemplate,
                       @Qualifier("shopLocalCache") Cache<Long, Shop> shopLocalCache,
                       RedissonBloomFilter<Long> shopIdBloomFilter) {
        this.shopMapper = shopMapper;
        this.redisTemplate = redisTemplate;
        this.shopLocalCache = shopLocalCache;
        this.shopIdBloomFilter = shopIdBloomFilter;
    }

    /**
     * 多级缓存查询商户详情
     * @param shopId 商户ID
     * @return 商户详情
     */
    public Shop queryShopById(Long shopId) {
        // 1. 先查本地缓存（Caffeine），命中直接返回，最快（毫秒级）
        Shop shop = shopLocalCache.getIfPresent(shopId);
        if (shop != null) {
            return shop;
        }

        // 2. 本地缓存未命中，查布隆过滤器，判断商户ID是否存在（防止缓存穿透）
        if (!shopIdBloomFilter.contains(shopId)) {
            // 无效ID，直接返回null，不查Redis和数据库
            return null;
        }

        // 3. 布隆过滤器判断存在，查Redis缓存
        String shopJson = redisTemplate.opsForValue().get(SHOP_REDIS_KEY + shopId);
        if (StrUtil.isNotBlank(shopJson)) {
            // Redis命中，反序列化为Shop对象
            shop = JSONUtil.toBean(shopJson, Shop.class);
            // 回写本地缓存，提升下次查询速度
            shopLocalCache.put(shopId, shop);
            return shop;
        }

        // 4. Redis未命中，查数据库（兜底）
        shop = shopMapper.selectById(shopId);
        if (shop == null) {
            // 数据库也未命中，返回null（可选：缓存空值，进一步防止缓存穿透）
            return null;
        }

        // 5. 数据库命中，将数据写入Redis和本地缓存，设置动态TTL（防缓存雪崩）
        // 动态TTL：30-40分钟随机，避免大量key同时过期
        long ttl = 30 * 60 + RandomUtil.randomInt(0, 600);
        redisTemplate.opsForValue().set(SHOP_REDIS_KEY + shopId, JSONUtil.toJsonStr(shop), ttl, TimeUnit.SECONDS);
        shopLocalCache.put(shopId, shop);

        return shop;
    }

    // 原有业务方法：查询商品实时库存（供MCP工具调用）
    public Integer getRealStock(Long shopId, Long goodsId) {
        // 模拟查询库存（实际项目中从数据库/Redis查询）
        return 100; // 示例值，可替换为真实逻辑
    }
}
```

### 3.2.4 技术栈体现（面试重点讲）

- Caffeine：高性能本地缓存，理解本地缓存的适用场景，体现缓存分层设计能力

- Redis：分布式缓存，掌握缓存写入、读取、过期时间设置，体现分布式缓存应用能力

- Redisson：布隆过滤器、分布式锁（后续秒杀优化用），体现分布式组件应用能力

- 缓存设计：多级缓存流程、缓存穿透/雪崩的解决方案，体现性能优化和问题解决能力

### 3.2.5 面试亮点

能清晰讲清多级缓存的流程和设计思路；能区分本地缓存（Caffeine）和分布式缓存（Redis）的适用场景；能详细说明缓存穿透、雪崩的解决方案，并有实际代码落地，而非纯理论。

## 优化3：优惠券秒杀高并发优化（Redisson分布式锁+消息队列）

### 3.3.1 业务场景

黑马点评的优惠券秒杀场景（如9.9元奶茶券、满200减100券），存在高并发请求（QPS可达数千），原有秒杀功能用简单Redis锁，存在集群环境下锁失效、超卖、同步下单阻塞等问题。优化为「Redisson分布式锁+消息队列异步下单」，提升并发能力，解决超卖、锁失效等问题。

### 3.3.2 优化目的

- 覆盖Java后端高并发核心考点：分布式锁、消息队列、异步编程、幂等性处理

- 体现高并发场景的解决方案设计能力，面试时能讲清「如何应对高并发秒杀」

- 贴合电商/本地生活平台的核心业务场景，提升项目的实战性和面试竞争力

### 3.3.3 实现步骤（详细可落地，1-2小时搞定）

#### 步骤1：引入消息队列依赖（Redis List，简单易实现，不用额外部署RabbitMQ）

用Redis List实现简单的消息队列，适合实习生项目，面试时可讲「若并发更高，可替换为RabbitMQ/RocketMQ」，体现技术选型能力。

```xml
<!-- 已引入Redis依赖，无需额外引入其他依赖，直接用Redis List实现消息队列 -->

```

#### 步骤2：改造秒杀接口，用Redisson分布式锁替换简单Redis锁

```java

@Service
public class SeckillService {
    private final RedissonClient redissonClient;
    private final StringRedisTemplate redisTemplate;
    private final CouponMapper couponMapper;
    private final OrderMapper orderMapper;

    // 构造器注入依赖
    public SeckillService(RedissonClient redissonClient, StringRedisTemplate redisTemplate,
                          CouponMapper couponMapper, OrderMapper orderMapper) {
        this.redissonClient = redissonClient;
        this.redisTemplate = redisTemplate;
        this.couponMapper = couponMapper;
        this.orderMapper = orderMapper;
    }

    // Redis key定义（秒杀相关）
    private static final String SECKILL_STOCK_KEY = "seckill:stock:"; // 秒杀库存key
    private static final String SECKILL_ORDER_KEY = "seckill:order:"; // 用户下单标记key
    private static final String SECKILL_QUEUE_KEY = "seckill:queue"; // 秒杀消息队列key

    /**
     * 优惠券秒杀接口（高并发优化版）
     * 核心流程：Redis预扣库存 → Redisson分布式锁 → 双重校验 → 消息队列异步下单
     */
    public Result seckillVoucher(Long voucherId, Long userId) {
        // 1. 先查Redis，预校验库存和用户是否已下单（减少数据库压力）
        if (!checkStockAndUser(voucherId, userId)) {
            return Result.fail("库存不足或已下单");
        }

        // 2. 获取Redisson分布式锁（锁key：按优惠券ID分锁，提升并发度）
        RLock lock = redissonClient.getLock("seckill:lock:" + voucherId);
        try {
            // 尝试获取锁：等待0秒，持有10秒，防止死锁（看门狗机制会自动续期）
            boolean isLocked = lock.tryLock(0, 10, TimeUnit.SECONDS);
            if (!isLocked) {
                // 未获取到锁，返回系统繁忙
                return Result.fail("系统繁忙，请稍后再试");
            }

            // 3. 拿到锁后，双重校验（防止并发情况下的超卖）
            if (!checkStockAndUser(voucherId, userId)) {
                return Result.fail("库存不足或已下单");
            }

            // 4. Redis预扣库存（原子操作，防止超卖）
            Long stock = redisTemplate.opsForValue().decrement(SECKILL_STOCK_KEY + voucherId);
            if (stock < 0) {
                // 库存不足，回滚预扣的库存
                redisTemplate.opsForValue().increment(SECKILL_STOCK_KEY + voucherId);
                return Result.fail("库存不足");
            }

            // 5. 标记用户已下单（Redis set，防止重复下单）
            redisTemplate.opsForValue().set(SECKILL_ORDER_KEY + voucherId + ":" + userId, "1", 24, TimeUnit.HOURS);

            // 6. 把下单消息写入Redis消息队列，异步处理数据库写入（解耦，提升并发）
            SeckillMessage message = new SeckillMessage(voucherId, userId);
            redisTemplate.opsForList().rightPush(SECKILL_QUEUE_KEY, JSONUtil.toJsonStr(message));

            // 7. 直接返回秒杀成功，无需等待数据库写入
            return Result.ok("秒杀成功，订单处理中");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return Result.fail("系统错误");
        } finally {
            // 8. 释放锁（仅释放当前线程持有的锁，防止误释放）
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

    /**
     * 预校验：库存是否充足 + 用户是否已下单
     */
    private boolean checkStockAndUser(Long voucherId, Long userId) {
        // 1. 查库存
        String stockStr = redisTemplate.opsForValue().get(SECKILL_STOCK_KEY + voucherId);
        if (StrUtil.isBlank(stockStr) || Integer.parseInt(stockStr) <= 0) {
            return false;
        }
        // 2. 查用户是否已下单
        String orderFlag = redisTemplate.opsForValue().get(SECKILL_ORDER_KEY + voucherId + ":" + userId);
        return StrUtil.isBlank(orderFlag);
    }

    /**
     * 消息队列监听器：异步处理订单写入数据库（独立线程，不阻塞秒杀接口）
     */
    @Component
    public class SeckillQueueListener {
        @Scheduled(fixedRate = 100) // 每100毫秒拉取一次消息
        public void processSeckillQueue() {
            // 1. 从消息队列拉取消息（左弹出，FIFO）
            String messageJson = redisTemplate.opsForList().leftPop(SECKILL_QUEUE_KEY);
            if (StrUtil.isBlank(messageJson)) {
                return; // 无消息，直接返回
            }

            // 2. 解析消息
            SeckillMessage message = JSONUtil.toBean(messageJson, SeckillMessage.class);
            Long voucherId = message.getVoucherId();
            Long userId = message.getUserId();

            // 3. 异步写入数据库（订单表、优惠券库存表）
            try {
                // 3.1 扣减数据库中的优惠券库存
                Coupon coupon = couponMapper.selectById(voucherId);
                if (coupon == null || coupon.getStock() <= 0) {
                    return;
                }
                coupon.setStock(coupon.getStock() - 1);
                couponMapper.updateById(coupon);

                // 3.2 生成订单
                Order order = new Order();
                order.setUserId(userId);
                order.setVoucherId(voucherId);
                order.setStatus(0); // 0：待核销
                order.setCreateTime(LocalDateTime.now());
                orderMapper.insert(order);
            } catch (Exception e) {
                // 异常处理：回滚Redis库存和下单标记（简单容错）
                redisTemplate.opsForValue().increment(SECKILL_STOCK_KEY + voucherId);
                redisTemplate.delete(SECKILL_ORDER_KEY + voucherId + ":" + userId);
                e.printStackTrace();
            }
        }
    }

    // 秒杀消息实体类
    @Data
    public static class SeckillMessage {
        private Long voucherId;
        private Long userId;

        public SeckillMessage(Long voucherId, Long userId) {
            this.voucherId = voucherId;
            this.userId = userId;
        }
    }
}
    
```

#### 步骤3：秒杀库存预热（项目启动时，将数据库优惠券库存同步到Redis）

```java

// 加入之前的BloomFilterInitializer类，新增库存预热逻辑
@Component
public class BloomFilterInitializer implements CommandLineRunner {
    // 新增依赖
    private final CouponMapper couponMapper;
    private final StringRedisTemplate redisTemplate;

    // 完善构造器
    public BloomFilterInitializer(RedissonBloomFilter<Long> shopIdBloomFilter, ShopMapper shopMapper,
                                  CouponMapper couponMapper, StringRedisTemplate redisTemplate) {
        this.shopIdBloomFilter = shopIdBloomFilter;
        this.shopMapper = shopMapper;
        this.couponMapper = couponMapper;
        this.redisTemplate = redisTemplate;
    }

    @Override
    public void run(String... args) throws Exception {
        // 1. 布隆过滤器预热（原有逻辑）
        // ...（省略原有代码）

        // 2. 秒杀库存预热：将数据库中的秒杀优惠券库存同步到Redis
        List<Coupon> seckillCoupons = couponMapper.selectList(new QueryWrapper<Coupon>().eq("is_seckill", 1));
        for (Coupon coupon : seckillCoupons) {
            redisTemplate.opsForValue().set(SECKILL_STOCK_KEY + coupon.getId(), coupon.getStock().toString());
        }
        System.out.println("秒杀库存预热完成，共加载优惠券：" + seckillCoupons.size() + "个");
    }
}
    
```

### 3.3.4 技术栈体现（面试重点讲）

- Redisson分布式锁：掌握分布式锁的实现、看门狗机制、锁释放逻辑，解决集群环境下的锁失效问题

- 消息队列（Redis List）：理解异步编程思想，用消息队列解耦秒杀接口和数据库写入，提升并发能力

- 高并发处理：预校验、双重校验、Redis原子操作，解决超卖、重复下单问题

- 幂等性处理：用Redis set标记用户下单状态，用数据库唯一索引（userId+voucherId）防止重复订单

### 3.3.5 面试亮点

能讲清高并发秒杀的完整解决方案，包括分布式锁的选型（Redisson vs 简单Redis锁）、消息队列的作用、超卖和重复下单的解决思路；能量化优化效果（如“并发能力从几百QPS提升到3000+ QPS”），体现工程落地能力。

## 优化4：流式接口实现（Spring WebFlux）

### 3.4.1 业务场景

AI客服接口若返回内容较长（如详细的商户介绍、优惠券规则），同步接口会出现超时问题，用户体验差。用Spring WebFlux实现SSE流式接口，让AI的回复实时推送给前端，一个字一个字显示，提升交互体验。

### 3.4.2 优化目的

- 覆盖Java后端进阶考点：响应式编程、Spring WebFlux、SSE流式响应

- 体现接口优化能力，面试时能讲清「流式接口与普通同步接口的区别」

- 贴合AI对话场景，让项目更具实战性，区别于普通后端项目

### 3.4.3 实现步骤（详细可落地，1小时搞定）

#### 步骤1：引入Spring WebFlux依赖

```xml
<!-- Spring WebFlux（响应式编程、流式接口） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
    
```

#### 步骤2：改造AI客服Service，实现流式响应

```java
@Service
public class ShopAICustomerService {
    // 原有依赖和方法不变，新增流式方法
    private final ChatClient chatClient;
    private final StringRedisTemplate redisTemplate;

    // 构造器不变
    // ...（省略原有代码）

    /**
     * 流式AI客服方法：基于Spring WebFlux，返回Flux<String>，实现SSE流式响应
     */
    public Flux<String> streamAskShopAI(Long shopId, String userQuestion, String userId) {
        // 1. 检索RAG知识库、获取会话记忆（和同步接口逻辑一致）
        String shopKnowledge = getShopKnowledge(shopId);
        String chatHistory = getChatHistory(userId);
        String prompt = buildPrompt(shopKnowledge, chatHistory, userQuestion);

        // 2. 调用Spring AI的流式API，返回Flux<String>（每一个元素是AI的一段回复）
        Flux<String> responseFlux = chatClient.prompt()
                .user(prompt)
                .stream() // 开启流式响应
                .content() // 只获取响应内容，忽略其他信息
                .doOnComplete(() -> {
                    // 3. 流式响应完成后，保存会话记忆（异步执行，不阻塞流式响应）
                    String fullResponse = responseFlux.collectList().block().stream().collect(Collectors.joining());
                    saveChatHistory(userId, userQuestion, fullResponse);
                })
                .onErrorResume(e -> {
                    // 4. 异常处理：返回错误信息，不中断流式响应
                    return Flux.just("抱歉，AI客服出现异常，请稍后再试");
                });

        return responseFlux;
    }

    // 原有方法（askShopAI、getShopKnowledge等）不变
    // ...（省略原有代码）
}
    
```

#### 步骤3：编写流式接口（Controller层）

```java

@RestController
@RequestMapping("/api/ai/customer")
public class ShopAIController {
    private final ShopAICustomerService aiCustomerService;

    public ShopAIController(ShopAICustomerService aiCustomerService) {
        this.aiCustomerService = aiCustomerService;
    }

    // 原有同步接口不变
    // ...（省略同步接口代码）

    /**
     * 流式AI客服接口（核心）
     * produces = MediaType.TEXT_EVENT_STREAM_VALUE：指定返回SSE流式响应
     */
    @GetMapping(value = "/stream-ask", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamAskAI(
            @RequestParam Long shopId,
            @RequestParam String userQuestion,
            @RequestParam String userId
    ) {
        // 直接返回Flux<String>，Spring WebFlux自动处理流式响应
        return aiCustomerService.streamAskShopAI(shopId, userQuestion, userId);
    }
}
    
```

#### 步骤4：简单前端测试（可选，面试时可讲）

用简单的HTML/JS写一个对话窗口，调用流式接口，实现实时接收AI回复的效果（不用复杂样式，能演示即可）。

```html
<!DOCTYPE html>
黑马点评AI客服用户：${userQuestion}AI：${aiResponse}AI：${aiResponse}AI：抱歉，出现异常，请稍后再试
```
> （注：文档部分内容可能由 AI 生成）