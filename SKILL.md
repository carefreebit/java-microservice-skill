---
name: java-microservice
description: 面向 Java 后端微服务的开发规范与 OpenSpec 驱动开发流程。当用户编写、评审、重构或设计 Spring Boot / Spring Cloud Alibaba / Dubbo 微服务代码，在已有框架或存量代码上继续开发，导入 PRD 分析需求，选择技术栈与版本，或需要阿里巴巴 Java 规约、分布式服务设计、脚手架、单元测试与集成测试指导时使用。Use when developing, reviewing, or designing Java microservices with Spring Boot / Spring Cloud Alibaba / Dubbo, adding features to an existing codebase, importing a PRD, selecting a stack, or applying Alibaba Java coding standards, distributed service design, scaffolding, unit and integration testing.
---

# Java 微服务开发规范

基于《阿里巴巴 Java 开发手册》与 Spring 生态的领域开发规范，覆盖编码、分布式设计、脚手架、工程结构、测试与存量项目接入，并通过 OpenSpec 管理从需求到交付的完整流程。

## 何时使用

- 编写、修改、评审 Java 后端微服务代码
- 在已有框架或存量代码上新增功能、模块或服务
- 从 PRD 提取需求、澄清歧义
- 选择或更换技术栈、组件与版本
- 设计服务拆分、分布式事务、幂等、消息、限流熔断等方案
- 生成分层脚手架与配套测试
- 排查 Java 微服务问题

不适用：纯前端、单机脚本、非 Java 技术栈。

## 流程门禁（必须遵守）

1. 启动时检查项目是否存在 `openspec/` 根目录。不存在时，告知用户并让其选择：初始化 OpenSpec，或仅做规范咨询；不得静默绕过。
2. 任何代码修改必须先有已批准的 OpenSpec change。没有 change 时，先走 propose 流程产出 proposal / specs / design / tasks。
3. 规划完成后停止并等待用户明确确认，不得在同一轮开始编码。
4. 进入 apply 后以 change 的 `tasks.md` 为唯一清单，逐项实施、验证、勾选。
5. 行为变更任务必须先写失败测试（RED），再实现（GREEN），并提交两次运行证据；纯重构使用 BASELINE/REGRESSION 证据。
6. 测试通过后才可勾选任务；全部完成并验证后建议归档。
7. 复用 `openspec` CLI 与 `openspec-*` skills 读取状态和归档，不创建并行的计划或进度文件。

### 编码前三问

修改任何项目代码前，先回答并在回复中写明：

1. 当前属于哪个 change？
2. 执行 `tasks.md` 中的哪一项？
3. 对应需求是否已经用户确认？

任一无法回答时，停止编码并回到需求确认或规划阶段。

### PRD 入口

用户提供 PRD（Markdown / Word / PDF）时：

1. 先读取并进入需求分析，不直接实现。
2. 按 `references/prd-analysis.md` 提取、分类、提问，输出确认表并写入 `openspec/changes/<change>/requirements.md`。
3. 用户确认范围后，才创建或补全 OpenSpec 规划。

## 存量项目入口

在已有框架或存量代码中开发时：

1. 检查 `openspec/project-profile.md`。不存在时，先加载 `references/brownfield-integration.md` 完成勘察并生成画像。
2. 画像存在时，核对版本、模块与既有约定；不一致先给 diff，用户确认后更新。
3. 选择流程档位：轻量（已画像模块的小改动）、标准（新增功能或模块）、完整（新建服务或更换中间件），并在规划中说明理由。
4. 优先复用现有统一响应、异常、错误码、基类、转换器与测试设施，不生成平行实现。
5. 修改无测试覆盖的老代码前，先建最小测试地基并补特征测试，取得 BASELINE 全绿后再改。

## 技术栈选择入口

需要确定组件或版本时：

1. 加载 `stacks/README.md`，按其说明展示或匹配预设。
2. 存量项目：探测实际版本，以项目为准，预设只作参考与差异提示。
3. 全新项目：询问用户自行选择、委托推荐还是自定义；用户委托时选 `status: recommended` 预设并说明理由。
4. 选择结果、实际版本、偏离项与理由写入 `openspec/project-profile.md` 的组件与版本决策记录。
5. 预设与自定义文件使用相对路径，可随整个 skill 文件夹复制到其他电脑。

## 任务路由

| 任务类型 | 加载文档 |
| --- | --- |
| PRD 需求分析 | `references/prd-analysis.md` |
| 存量项目接入 | `references/brownfield-integration.md`、`openspec/project-profile.md` |
| 技术栈选择 | `stacks/README.md`、对应预设文件 |
| 新增实现 | `references/alibaba-coding-standards.md`、`references/scaffolding-templates.md`、`references/scaffolding-test-templates.md`、`references/testing-standards.md`；存量项目加 `references/brownfield-integration.md` |
| 代码评审 | `references/alibaba-coding-standards.md`、`references/review-checklist.md`；存量项目加 `references/brownfield-integration.md` |
| 脚手架生成 | `references/scaffolding-templates.md`、`references/scaffolding-test-templates.md`、`references/testing-standards.md`；存量项目加 `references/brownfield-integration.md` |
| 分布式方案设计 | `references/microservice-architecture.md` |
| 问题排查 | `references/microservice-architecture.md`、`references/alibaba-coding-standards.md`；存量项目加 `references/brownfield-integration.md` |

渐进披露：只加载与当前任务直接相关的文档，不预先加载全部。

## 产出纪律

- 评审与整改结论必须注明规约类别与级别，如"并发规约【强制】"。
- 每处不合规给出可直接执行的修改建议，不罗列原文。
- 无法判定的项标为"待确认"，说明需要的信息，不虚构违规。
- 生成代码附最小验证命令；行为代码先测试后实现。
- 存量项目先说明复用了哪些现有设施，避免平行实现。
- 不把参考文档大段原文复制到回复中。

## 收尾自检

结束前按 `references/review-checklist.md` 自检，报告未解决项或明确声明无遗留违规，并附测试结果与 RED/GREEN 证据。纯评审或咨询等无代码变更的场景，说明证据不适用即可，不编造证据。

## 边界与澄清

- 项目版本与预设或推荐基线不一致时，以项目实际为准，并把偏离记录到画像。
- 全新项目先询问组件与版本选择，不替用户拍板。
- 需求缺少会改变设计的信息时先提问，不擅自假设。
- 不重构与当前任务无关的代码，不升级无关组件；范围外问题登记技术债。
