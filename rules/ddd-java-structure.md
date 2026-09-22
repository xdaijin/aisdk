# Java 后端 DDD 目录结构规范（全局强制）

> 创建任何后端 Java 新类之前，先按下表确定归属包；拿不准时遵循"放所属业务模块，别丢 common"。
> 依赖方向永远 `interfaces → application → domain ← infrastructure`（domain 零框架依赖）。

## 1. 模块划分（多模块项目）

```
xxx-common        共享内核（Result、BusinessException），零 Spring 依赖，宁可小不可杂
xxx-<域>          一个限界上下文一个模块
xxx-app           组装层：唯一可执行模块（启动类、Security、全局异常、application.yml）
```

铁律：`app → 业务模块 → common`，业务模块之间禁止互相依赖；跨上下文协作在 app 层编排或走 RPC/事件。

## 2. 业务模块内部四层

```
com.example.<域>/
│
├── interfaces/                                  # 接入层：所有入站适配器
│   ├── XxxController.java                       #   HTTP 入站
│   ├── consumer/                                #   MQ 消费者（入站，和 Controller 一样薄）
│   │   └── message/                             #     消费消息体（入站协议模型）
│   ├── job/                                     #   定时任务（入站）
│   ├── dto/                                     #   对外 HTTP 协议模型（record）
│   │   ├── XxxRequest.java                      #     允许序列化注解（脱敏/@JsonProperty）
│   │   └── XxxResponse.java                     #     application 禁止使用这里的类型
│   └── assembler/                               #   DTO ↔ Command/Result 转换（可选）
│
├── application/                                 # 应用层：用例编排、事务边界
│   ├── service/                                 #   应用服务（一个聚合一个）
│   │   ├── XxxAppService.java                   #     写用例：一个方法 = 一个用例 = 一个事务
│   │   └── XxxQueryService.java                 #     读用例（读写都重时拆，可选）
│   ├── command/                                 #   写用例入参（record，用例间不复用）
│   ├── query/                                   #   读用例查询条件
│   ├── result/                                  #   应用层返回对象（record；简单场景可省）
│   ├── assembler/                               #   对象转换（可选，推荐 MapStruct）
│   ├── listener/                                #   领域事件订阅，只做跨聚合编排（可选）
│   └── gateway/                                 #   出站端口（防腐层）
│       ├── XxxGateway.java                      #     接口：技术无关，禁出现 SDK 类型
│       └── dto/                                 #     端口契约出入参（纯 record，禁序列化注解）
│           ├── XxxParam.java
│           └── XxxResult.java
│
├── domain/                                      # 领域层：业务核心，零框架依赖
│   ├── model/                                   #   聚合根/实体/值对象/业务枚举常量
│   │   ├── Xxx.java                             #     聚合根：业务规则内聚（order.cancel()）
│   │   ├── Money.java                           #     值对象（有领域含义的入参也是它）
│   │   └── XxxStatus.java                       #     业务枚举
│   ├── event/                                   #   领域事件（不可变契约，与 model 平级）
│   ├── repository/                              #   仓储接口（只定义不实现）
│   │   └── query/                               #     仓储查询条件对象
│   └── service/                                 #   领域服务/领域能力接口
│       └── param/                               #     领域服务参数对象（无领域行为，仅打包）
│
└── infrastructure/                              # 基础设施层：技术实现细节
    ├── persistence/
    │   ├── mapper/                              #   ORM Mapper（@MapperScan 精确指向此包）
    │   │                                        #     只被 XxxRepositoryImpl 使用
    │   ├── po/                                  #   持久化对象（可选，模型与表分离时用）
    │   └── XxxRepositoryImpl.java               #   仓储实现（不进 @MapperScan 范围）
    ├── messaging/                               #   MQ 出站：生产者实现 + message/（出站消息体）
    ├── client/                                  #   外部服务客户端（实现 application/gateway）
    │   ├── XxxGatewayImpl.java                  #     负责 gateway/dto ↔ 第三方协议翻译
    │   └── <第三方服务>/dto/                     #     第三方 wire 格式（允许序列化注解）
    ├── config/                                  #   本模块的 @ConfigurationProperties
    └── util/                                    #   本模块专用技术工具（克制）
```

## 3. 分层红线

| 层 | 禁止 |
|---|---|
| interfaces | 业务判断；直接调 Mapper/RepositoryImpl；try-catch 包响应 |
| application | import interfaces/infrastructure；感知 Web 类型 |
| domain | 依赖 Spring/外层任何包；复用 application 的 Command |
| infrastructure | 承载业务规则；其类型泄漏到端口签名 |

### domain 注解规则（"零框架依赖"的具体含义）

禁的是**框架行为耦合**（Spring 类型进入业务代码、领域对象进容器、domain 管事务），不是注解本身。

- ✅ 允许：Lombok（编译期代码生成）、`jakarta.validation`（JSR-303 规范）、JDK 注解
- ⚠️ 容忍（务实取舍）：MP `@TableName`/`@TableId`、Jackson 注解——被动元数据，类不经框架照样可用
- ⚠️ 例外：**领域服务实现允许 `@Component`**（无状态单例，组件扫描注册，省 @Bean 样板）；依赖仍只能是 domain 的接口，业务代码中不得出现任何 Spring 类型
- ❌ 硬红线（不放松）：
  1. `model/` 聚合根/实体/值对象禁止任何容器注解——它们由仓储/工厂创建，不是 Bean
  2. 禁止 `@Autowired` 字段注入——一律 Lombok `@RequiredArgsConstructor` 构造器注入
  3. 禁止 `@Transactional`——事务边界永远属于 application 层
  4. 禁止直接使用 `ApplicationEventPublisher`——domain 定义自己的 `DomainEventPublisher` 接口，Spring 桥接实现放 infrastructure

## 4. 三套 DTO 互不复用

| 类 | 位置 | 表达 |
|---|---|---|
| XxxRequest/Response | interfaces/dto/ | 本系统对外 HTTP API |
| XxxParam/Result | application/gateway/dto/ | 出站端口契约（技术无关） |
| XxxApiRequest/Response | infrastructure/client/<服务>/dto/ | 第三方 wire 格式 |

转换链：Request →(assembler)→ Command →(application)→ Param →(GatewayImpl)→ ApiRequest

## 5. 快速决策

- 新类不知道放哪 → 查"速查表"：业务规则→domain/model 聚合方法；跨实体规则→domain/service；有含义的入参→domain/model 值对象；纯参数打包→对应层的 param/ 或 query/ 子包
- 常量/枚举 → 跟着所属聚合走（domain/model，枚举是聚合的属性）；**禁止建 domain/enums/ 集中目录**（诱导跨聚合复用）；枚举过多时允许 model/enums/ 子包，全项目统一一种风格；状态流转规则内聚到枚举方法里（如 canTransitTo）
- 配置属性→所属模块 infrastructure/config；全局配置类→app
- 异常 → `BusinessException`/`SystemException` 放 common（跨域原语）；域专属异常类型放 `<域>/domain/exception/` 且必须 extends BusinessException（用到才建）；业务错误码跟着域走按号段分配，禁止在 common 搞集中式错误码表；SDK/IO 异常在 infrastructure 包装成 SystemException，不许外泄；异常转响应只在 app 的 GlobalExceptionHandler
- 工具类 → 先问能否做成值对象/领域服务；真需要才放 infrastructure/util
- MQ 消费、定时任务 → 都是入站适配器，放 interfaces/consumer、interfaces/job
- 子包用到才建（无 MQ 不建 consumer/messaging）；单域 <10 类时 application 可扁平，10+ 再按类型分包
