# Stack Catalog

可选技术栈预设目录。每个预设固定一组兼容的 JDK、Spring 与组件版本，供新项目选型或存量项目参考。

## 目录结构

```text
stacks/
├── README.md          # 本文件
├── _template.yaml     # 自定义预设模板
├── greenfield/        # 全新项目
├── brownfield/        # 存量项目参考
└── custom/            # 用户自定义，随 skill 文件夹复制
```

## 怎么选

1. **存量项目**：先探测项目实际版本与组件，匹配最近的预设作为参考。以项目实际值为准，预设不要求升级。
2. **全新项目，用户自己选**：按场景浏览预设，选一个。
3. **全新项目，让 skill 决定**：skill 选择 `status: recommended` 的预设，并说明适用理由与取舍。
4. **没有匹配**：从 `_template.yaml` 复制一份到 `custom/`，填写实际组合；skill 会像内置预设一样读取。

选择结果、实际版本与偏离项写入 `openspec/project-profile.md` 的组件与版本决策记录。

## 字段说明

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `id` | 是 | kebab-case 唯一标识 |
| `name` | 是 | 展示名称 |
| `status` | 是 | `recommended` / `legacy` / `experimental` |
| `verified_at` | 是 | 最后核验日期，YYYY-MM-DD |
| `jdk` | 是 | JDK 主版本 |
| `language_level` | 是 | 源码级别，用于避免生成高版本语法 |
| `spring_boot` / `spring_cloud` / `spring_cloud_alibaba` | 是 | 版本线 |
| `bom` | 是 | `groupId:artifactId:version` 列表，version 可用 `${...}` 占位 |
| `components` | 是 | 注册中心、配置、网关、RPC、ORM、MQ、缓存、鉴权、可观测、测试 |
| `validation_namespace` | 是 | `jakarta` 或 `javax` |
| `test_framework` | 是 | `junit5` / `junit4` / 其他 |
| `choose_when` | 是 | 适用场景 |
| `avoid_when` | 是 | 不适用场景 |
| `constraints` | 是 | 使用约束与已知限制 |

## 自定义预设

- 放在 `stacks/custom/`，文件名建议 `<company>-<stack>.yaml`。
- 从 `_template.yaml` 复制，填完必填字段。
- 不使用绝对路径、机器地址、口令或账号；用占位符或引用项目配置。
- skill 更新或重新安装时，`custom/` 下的文件不会被覆盖。

## 复制到其他电脑

1. 复制整个 `java-microservice` 文件夹到目标机器的 `~/.claude/skills/`（或项目的 `.claude/skills/`）。
2. 内置预设与 `custom/` 会一起带走。
3. 在新机器上打开新会话，skill 会重新扫描目录；已运行的会话需要重启。

## 维护

- 每个预设的 `verified_at` 表示最后核对官方发布说明的日期。
- 版本过时只提示，不自动修改；需要更新时由用户或维护者改文件。
- 新增预设后，在本文件与 `SKILL.md` 的技术栈选择入口中确认可发现。
