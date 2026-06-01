## 1.创建CLAUDE.md
```markdown
# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 项目概述

**Sky Take Out** (天空外卖) 是一个中国外卖配送平台 — 基于 Spring Boot 2.7.3 后端与 Vue 2 + TypeScript 管理后台前端，通过 Nginx 部署。它提供两个 API 面：一个管理门户和一个面向用户的订单服务。

## 技术栈

- **后端**: Spring Boot 2.7.3, MyBatis 2.2.0, MySQL (Druid 连接池), Redis, Redisson 3.17.6, RabbitMQ, WebSocket, AOP (AspectJ), FastJSON 1.2.76, JWT (jjwt 0.9.1), PageHelper, Knife4j 3.0.2, 阿里云 OSS, 微信支付 SDK, Apache POI
- **前端 (管理后台)**: Vue 2.6, TypeScript 3.6 (class-component + property-decorator 模式), Element UI 2.12, ECharts 5.3, Vuex 3.1, Axios
- **前端 (用户端)**: 预构建的 SPA，支持 PWA，由 Nginx 静态提供服务
- **其他**: 百度地图地理编码 API, Knife4j (API 文档)

## 模块结构

```
sky-take-out/          # Maven 根项目 (pom)
├── sky-common/        # 共享工具类、常量、异常、JWT 工具、OSS 工具
├── sky-pojo/          # DTO、实体、VO（数据传输对象 / 实体 / 视图对象）
├── sky-server/        # Spring Boot 应用程序 — 控制器、服务、映射器、配置
└── sky-web/
    ├── nginx-1.20.2/  # Nginx (端口 8090) 用于提供静态文件及反向代理
    └── project-sky-admin-vue-ts/  # Vue 管理后台前端 (开发服务器端口 8888)
```

## 关键架构模式

### 两个 API 入口

- **管理 API** (`/admin/**`): 员工登录、菜品/分类/套餐管理、订单处理、报表、工作台仪表板。由 `JwtTokenAdminInterceptor` 保护（排除 `/admin/employee/login`）。
- **用户 API** (`/user/**`): 用户登录（微信）、浏览菜品/分类、购物车、下单/支付、地址簿。由 `JwtTokenUserInterceptor` 保护（排除 `/user/user/login` 和 `/user/shop/status`）。

### 网络流量

```
浏览器 → Nginx (:8090) → /api/*      → Spring Boot (:8081) /admin/*
                          → /user/*    → Spring Boot (:8081) /user/*
                          → /ws/*      → Spring Boot (:8081) /ws/* (WebSocket)
                          → /*         → 静态 SPA (html/sky/)

开发环境: Vue CLI (:8888) → /api 代理 → http://localhost:8080/admin
```

管理员 JWT token 在 `token` 头中发送；用户 JWT token 在 `authentication` 头中发送。两者的 token TTL 均为 2 小时。JWT 密钥默认为 `itcast` (管理) 和 `itheima` (用户) — 可在 application.yml 中配置。

### 请求流程

```
请求 → JWT 拦截器 (解析 token，设置 BaseContext ThreadLocal) → 控制器 → 服务 → 映射器 → 数据库
```

- `BaseContext` 使用 `ThreadLocal` 存储当前用户/员工 ID。在 `afterCompletion()` 中清理。
- 多个控制器具有相同的简单类名（例如 `admin.DishController` 和 `user.DishController`）。通过显式的 bean 名称区分：`@RestController("adminDishController")`、`@RestController("userDishController")` 等。

### JSON 序列化

- **HTTP 消息转换**: 自定义 `JacksonObjectMapper` 处理 Java 8 时间类型 (`LocalDateTime` → `"yyyy-MM-dd HH:mm"`，`LocalDate` → `"yyyy-MM-dd"`，`LocalTime` → `"HH:mm:ss"`)。忽略未知属性。
- **业务逻辑**: 代码中使用**阿里巴巴 FastJSON** (`JSON.parseObject()`、`JSON.toJSONString()`、`JSONObject`、`JSONArray`) — 而不是 Jackson — 在 `OrderServiceImpl`、`PayNotifyController`、`HttpClientUtil` 及其他服务中。

### AutoFill AOP

`@AutoFill` 注解（位于 sky-server，`com.sky.annotation.AutoFill`）+ `AutoFillAspect` 在实体插入/更新的 Mapper 方法上自动填充 `createTime`、`createUser`、`updateTime`、`updateUser`。使用反射调用 `AutoFillConstant` 中常量标识的 setter 方法。

### 订单状态机

订单依次经过：`待付款(1)` → `待确认(2)` → `已确认(3)` → `配送中(4)` → `已完成(5)`。已取消的订单变为 `已取消(6)`。

**关键规则**:
- 用户侧取消仅允许状态为 1 或 2 时（`status <= TO_BE_CONFIRMED`）。
- `reminder()` 仅对 `配送中(4)` 状态的订单触发。
- 支付状态：`未支付 = 0`，`已支付 = 1`，`已退款 = 2`。

### 双重超时取消机制

未支付订单通过**两条冗余路径**自动取消：

1. **RabbitMQ 死信队列**: 订单加入队列并设置 15 分钟 TTL；过期消息由 `OrderMessageListener` 消费并取消订单。
2. **定时任务备用方案** (`OrderTask`): `@Scheduled(cron = "0 * * * * ?")` 每分钟运行一次，检查超过 15 分钟的 `待付款` 订单。此外，`@Scheduled(cron = "0 0 1 * * ?")` 每天凌晨 1 点自动完成超过 60 分钟的 `配送中` 订单（使用批量 `UPDATE` SQL）。

### WebSocket 推送

`WebSocketServer`（端点 `ws://host:port/ws/{sid}`）在一个静态 `HashMap<String, Session>` 中管理会话。`sendToAllClient()` 向所有连接的客户端广播。在 `OrderServiceImpl` 中通过 `TransactionSynchronizationManager.afterCommit()` 注册推送通知，确保在通知之前数据库提交已完成。

### 手动 Redis 缓存与 Redisson 锁

尽管 `SkyApplication` 上标注了 `@EnableCaching`，但**没有使用 Spring 缓存注解**（`@Cacheable` 等）。所有缓存都是通过 `redisTemplate.opsForValue().get/set` 在菜品和店铺控制器中手动实现的。

面向用户的 `DishController` 使用 **Redisson 分布式锁** (`lock:dish_` + categoryId) 防止读穿透时的缓存击穿。

管理端 `DishController` 中的缓存失效使用 **Redis SCAN**（非 KEYS）并匹配模式 `dish_*`，以避免阻塞 Redis 单线程。

### Knife4j 兼容性补丁

`WebMvcConfiguration` 包含一个 `BeanPostProcessor`，为 Spring Boot 2.6+ 打补丁以兼容 Springfox — 它会过滤掉那些设置了 `PatternParser` 的处理程序映射。这解决了著名的 Springfox/Spring Boot 2.6 路径模式匹配冲突。同时，application.yml 中也配置了 `spring.mvc.pathmatch.matching-strategy: ant_path_matcher`。

### 前端关键模式

- **基于类的 Vue 组件**: 使用 `vue-class-component` + `vue-property-decorator`，配合 `@Component`、`@Prop`、`@Watch`。
- **使用装饰器的 Vuex**: 模块使用 `vuex-module-decorators`（`@Module`、`@Mutation`、`@Action`）。通过 `getModule(User)` 获得直接的基于类的访问。
- **重复请求去重**: Axios 请求拦截器对 `url + method + data` 进行 MD5 哈希，并通过 `CancelToken` 取消相同的进行中请求。
- **认证守卫**: `beforeEach` 路由守卫检查是否存在 `token` cookie。标记为 `meta.notNeedAuth: true` 的公共路由是唯一跳过检查的路由。
- **路由布局**: 所有管理路由都是根 `/` 的 `Layout` 组件的子路由（侧边栏 + 导航栏 + 内容区域）。隐藏路由（例如 `/dish/add`）使用 `meta.hidden: true` 将其从侧边栏中隐藏。

### 支付

微信支付集成：`OrderServiceImpl.payment()` 中的 `pay()` 调用是**桩代码**（被注释掉，状态硬更新）。`PayNotifyController` 包含使用 `AesUtil` 的完整解密逻辑，因此回调端点是真实的 — 但支付发起在当前代码中是模拟的。

### 配送范围检查

`OrderServiceImpl.checkOutOfRange()` 使用百度地图地理编码 API v3 和轻量方向 API v1。商家地址硬编码为 `"江苏省连云港市海州区花果山街道江苏海洋大学苍梧校区"`。距离阈值：5000 米。

## 常用命令

### 后端 (Maven)

所有 Maven 命令必须从 `sky-take-out/` 目录（父 POM）运行，而不是仓库根目录。

```bash
cd sky-take-out

# 构建所有模块（跳过测试）
mvn clean package -DskipTests

# 启动服务器
mvn spring-boot:run -pl sky-server

# 使用特定 profile 运行
mvn spring-boot:run -pl sky-server -Dspring.profiles.active=dev

# 运行单个测试
mvn test -pl sky-server -Dtest=ClassName#methodName

# 仅编译
mvn compile -pl sky-server
```

### 前端 (Vue 管理后台)

```bash
cd sky-take-out/sky-web/project-sky-admin-vue-ts

# 安装依赖
yarn install

# 开发服务器（运行在端口 8888，将 /api 代理到 http://localhost:8080/admin）
yarn serve

# 生产构建
yarn build
```

### Nginx

```bash
# 配置文件位于 sky-take-out/sky-web/nginx-1.20.2/conf/nginx.conf
# 启动 (Windows)
sky-take-out/sky-web/nginx-1.20.2/nginx.exe

# 重新加载配置
sky-take-out/sky-web/nginx-1.20.2/nginx.exe -s reload
```

## 配置

- `application.yml` 使用 `${sky.*}` 占位符。值通过 `application-{profile}.yml` 或环境变量解析。
- `application-dev.yml` 包含实际的开发凭据（数据库、Redis、RabbitMQ、阿里云 OSS、微信支付），并已提交到 git — **请不要在此处提交生产环境密钥**。
- API 文档可在 `http://localhost:8081/doc.html` 访问 (Knife4j)。

## 关键文件说明

- `SkyApplication.java` — Spring Boot 入口点 (`@EnableScheduling`、`@EnableCaching`)
- `WebMvcConfiguration.java` — MVC 配置、JWT 拦截器注册、Knife4j/Swagger 兼容性补丁、Jackson 定制
- `AutoFillAspect.java` — 用于在 Mapper 方法上自动填充审计字段的 AOP
- `OrderServiceImpl.java` — 核心订单逻辑（提交、支付、取消、状态转换、配送范围检查）
- `OrderMessageListener.java` — RabbitMQ 死信队列监听器，用于订单超时取消
- `OrderTask.java` — 订单超时取消和每日自动完成的定时任务备用方案
- `WebSocketServer.java` — 用于实时管理端通知的 WebSocket 服务器
- `WebSocketTask.java` — 包含一个被注释掉的心跳调度器
- `GlobalExceptionHandler.java` — 统一异常处理 → `Result` 错误响应（处理 `BaseException` 和 `SQLIntegrityConstraintViolationException`）
- `RedisConfiguration.java` — 创建一个 `RedisTemplate<String, Object>` bean，键使用 `StringRedisSerializer`
- `JacksonObjectMapper.java` — 用于 HTTP 消息中 Java 8 时间序列化的自定义 ObjectMapper

<!-- gitnexus:start -->
# GitNexus — 代码智能

本项目已由 GitNexus 索引为 **sky-take-out**（4288 个符号，8175 个关系，268 个执行流）。使用 GitNexus MCP 工具来理解代码、评估影响并安全地导航。

> 如果任何 GitNexus 工具警告索引已过时，请先在终端中运行 `npx gitnexus analyze`。

## 务必做到

- **在编辑任何符号之前，必须先运行影响分析。** 在修改函数、类或方法之前，运行 `gitnexus_impact({target: "symbolName", direction: "upstream"})` 并向用户报告爆炸半径（直接调用方、受影响的流程、风险级别）。
- **在提交之前必须运行 `gitnexus_detect_changes()`** 以验证你的变更只影响预期的符号和执行流。
- **如果影响分析返回高风险或严重风险，必须警告用户**，然后再进行编辑。
- 当探索不熟悉的代码时，使用 `gitnexus_query({query: "concept"})` 来查找执行流，而不是使用 grep。它会返回按流程分组并按相关性排序的结果。
- 当你需要特定符号的完整上下文（调用方、被调用方、参与的执行流）时，使用 `gitnexus_context({name: "symbolName"})`。

## 绝对禁止

- 绝不在未先运行 `gitnexus_impact` 的情况下编辑函数、类或方法。
- 绝不要忽略影响分析中的高风险或严重风险警告。
- 绝不要使用查找替换来重命名符号 — 使用 `gitnexus_rename`，它能理解调用图。
- 绝不要在不运行 `gitnexus_detect_changes()` 检查受影响范围的情况下提交变更。

## 资源

| 资源 | 用途 |
|------|------|
| `gitnexus://repo/sky-take-out/context` | 代码库概览，检查索引新鲜度 |
| `gitnexus://repo/sky-take-out/clusters` | 所有功能区域 |
| `gitnexus://repo/sky-take-out/processes` | 所有执行流 |
| `gitnexus://repo/sky-take-out/process/{name}` | 逐步执行轨迹 |

## CLI

| 任务 | 读取此技能文件 |
|------|----------------|
| 理解架构 / “X 是如何工作的？” | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| 爆炸半径 / “如果我修改 X，什么会坏？” | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| 追踪 bug / “为什么 X 失败了？” | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| 重命名 / 提取 / 拆分 / 重构 | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| 工具、资源、模式参考 | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| 索引、状态、清理、wiki CLI 命令 | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |

<!-- gitnexus:end -->
```