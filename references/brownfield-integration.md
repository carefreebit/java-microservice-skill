# 存量项目接入

用于在已有框架或存量代码上继续开发。核心原则：先勘察、再增量；项目既有约定优先；只改当前范围。

## 一、什么时候用

- 项目已有框架、脚手架或历史代码，不是从零开始
- 需要在现有模块中新增功能
- 需要新建服务，但要沿用公司既有技术栈
- 需要判断某个改动会不会影响现有调用方

## 二、流程分三档

| 档位 | 适用 | 最小产出 |
| --- | --- | --- |
| 轻量 | 已画像模块的小改动 | 复用画像、简短影响说明、触碰行为的测试 |
| 标准 | 新增功能或模块 | 刷新画像、完整影响分析、必要时特征测试 |
| 完整 | 新建服务或更换中间件 | 完整勘察、组件决策记录、迁移与回滚方案 |

在 proposal 或 design 中写明所选档位与理由，不要默认走完整流程。

## 三、接入勘察

按顺序检查，能自动获取的用现有命令，不新增脚本。

### 3.1 构建与版本

```bash
mvn help:evaluate -Dexpression=project.build.sourceEncoding -q -DforceStdout
mvn help:evaluate -Dexpression=maven.compiler.release -q -DforceStdout
mvn dependency:tree -Dverbose
```

- 父 POM、BOM、JDK 与编译级别
- Spring Boot、Spring Cloud、Spring Cloud Alibaba 版本
- 是否多模块、模块依赖方向

### 3.2 结构与约定

- 包结构与分层：Controller / Service / Mapper 的位置
- 统一响应、错误码、全局异常、参数校验
- DTO/VO/DO/BO 命名与转换器
- 日志、traceId、鉴权、拦截器
- 是否有公司自研 starter、基类、代码生成器

### 3.3 持久层与中间件

- MyBatis / MyBatis-Plus / JPA
- Mapper XML 位置、分页方式、迁移工具
- Nacos / Consul、MQ、缓存、事务
- 配置外置方式与 profile

### 3.4 测试栈

- JUnit 4 还是 5，断言库
- MockMvc / WebTestClient
- Testcontainers 或共享测试库
- 测试基类、命名习惯、是否可运行

### 3.5 安全边界

- 不记录口令、令牌、私钥、内部地址的具体值
- 画像只写配置项名称与来源，例如 `spring.datasource.password ← 环境变量`
- 不把项目文件上传到外部服务

## 四、项目画像

路径：`openspec/project-profile.md`。按仓库保存，模块级信息分节。

```markdown
# 项目画像

## 概览
- 仓库：example/order-platform
- 本次范围：order-service
- 最后核验：2026-11-12

## 构建与版本
- JDK：8
- Spring Boot：2.7.x
- Spring Cloud：2021.0.x
- Spring Cloud Alibaba：2021.0.5.x
- 父 POM：example-platform-parent

## 模块地图
| 模块 | 职责 | 依赖 |
| --- | --- | --- |

## 分层与包结构
- Controller：com.example.order.api
- Service：com.example.order.application
- Mapper：com.example.order.infrastructure

## 横切设施
| 设施 | 现有实现 | 复用方式 |
| --- | --- | --- |
| 统一响应 | Result<T> | 直接使用 |
| 业务异常 | BizException + ErrorCode | 沿用现有错误码段 |

## 持久层
- ORM：MyBatis-Plus 3.x
- Mapper XML：src/main/resources/mapper
- 迁移：Flyway

## 中间件
- 注册/配置：Nacos
- MQ：RocketMQ
- 缓存：Redis

## 测试栈
- JUnit 4
- MockMvc
- 共享测试库，无 Testcontainers

## 既有约定
- 命名、DTO/VO/DO、转换器、代码生成器

## 规约冲突表
| 项目约定 | 阿里规约 | 结论 |
| --- | --- | --- |

## 组件与版本决策记录
| 日期 | 背景 | 结论 | 理由 | 批准人 |
| --- | --- | --- | --- | --- |

## 技术债清单
| 位置 | 问题 | 影响 | 建议 |
| --- | --- | --- | --- |

## 核验记录
| 日期 | 核验人 | 变化 |
| --- | --- | --- |
```

### 画像维护

- 每个 change 开始时读取并核对；不一致时先给出 diff。
- 用户确认后更新，并记录核验日期。
- 不覆盖用户手写内容；冲突时提出讨论。
- 多模块仓库只详细记录本次涉及的模块。

## 五、既有设施复用

生成代码前先查下表，存在就引用，缺失才征求同意后创建。

| 设施 | 先查什么 | 复用方式 |
| --- | --- | --- |
| 统一响应 | `Result` / `ApiResponse` / `R` | 直接返回现有类型 |
| 业务异常 | `BizException` / `ServiceException` | 抛现有异常，使用现有错误码 |
| 错误码 | `ErrorCode` / `ResultCode` | 在既有模块段内新增 |
| 基类 | `BaseController` / `BaseService` | 继承或组合 |
| 转换器 | MapStruct / 自定义 Converter | 沿用现有方式 |
| 测试基类 | `BaseTest` / `BaseIT` | 继承现有基类 |

不做的事：

- 不生成与现有类型同名或功能重复的新类
- 不混入第二套 ORM、日志、映射或测试框架
- 不为了"符合模板"重排现有包结构

## 六、影响分析

在 proposal 或 design 中列出：

- 受影响模块与服务
- 对外接口与契约：新增、修改、废弃
- 表结构：DDL、索引、数据迁移、回滚
- 配置：新增项、默认值、环境差异
- 缓存：key、过期、失效策略
- 消息：topic、tag、消息键、重试与死信
- 定时任务与批处理
- 测试范围与回归清单
- 发布顺序、灰度与回滚点

跨服务改动必须写明调用方与兼容策略。

## 七、测试：地基、特征测试与证据

### 7.1 最小测试地基

项目没有可运行测试时，先补：

1. 与项目版本兼容的测试依赖
2. 一个最小测试基类或配置
3. 一个冒烟测试，确认能跑

### 7.2 特征测试

特征测试锁定现状，不表达期望：

1. 选出将被触碰的行为
2. 写测试覆盖现有输入与输出
3. 运行得到 BASELINE 全绿
4. 修改代码
5. 同一批测试仍全绿（REGRESSION）

只覆盖本次触碰的行为，不补全模块。

### 7.3 证据格式

```text
BASELINE
命令：mvn -q -Dtest=OrderAmountCalculatorTest test
结果：Tests run: 6, Failures: 0, Errors: 0

REGRESSION
命令：mvn -q -Dtest=OrderAmountCalculatorTest test
结果：Tests run: 6, Failures: 0, Errors: 0

RED
命令：...（新行为测试）
结果：FAILED — 期望新规则，实际仍为旧规则

GREEN
命令：...
结果：Tests run: 8, Failures: 0, Errors: 0
```

纯重构只需要 BASELINE 与 REGRESSION；行为变化才需要 RED 与 GREEN。

## 八、版本差异适配

只覆盖以下六类，未知组合停下来问。

| 差异 | 存量项目做法 |
| --- | --- |
| Java 语言级别 | JDK 8 不生成 record、var、text block |
| `javax` / `jakarta` | Boot 2 用 `javax.validation`、`javax.servlet` |
| Boot 2/3 配置 | 沿用 `bootstrap.yml`、`spring.factories` 或 Boot 3 的 `AutoConfiguration.imports` |
| JUnit 4/5 | 沿用项目测试框架，不引入第二套 |
| Swagger 2 / OpenAPI 3 | 沿用现有注解体系 |
| MyBatis / MyBatis-Plus | 沿用现有 Mapper、分页与条件构造方式 |

无法判断时：

- 标注"待确认"
- 向用户询问兼容性
- 不宣称已验证

## 九、范围控制与技术债

- 只改当前 change 范围内的代码。
- 发现范围外问题：记录到画像的技术债清单，不顺手改。
- 对外契约保持向后兼容；破坏性变更需用户明确同意并给出迁移方案。
- 重构、升级、换中间件作为独立 change，不混入业务改动。

## 十、组件引入决策

存量项目新增组件前，依次评估：

1. 现有组件是否够用
2. 运维成本与团队熟悉度
3. 许可、安全与合规
4. 可观测性与故障处理
5. 迁移与退出成本

结论写入画像的组件与版本决策记录，包含日期、背景、备选、结论、影响与批准人。

## 十一、输出格式

接入完成后，回复应包含：

1. 所选流程档位与理由
2. 画像路径与本次更新内容
3. 现有设施复用清单
4. 影响分析摘要
5. 测试策略与证据要求
6. 待确认问题
