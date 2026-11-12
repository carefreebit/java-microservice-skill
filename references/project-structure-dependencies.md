# 工程结构与依赖管理

本文档定义多模块工程的结构边界、版本基线与依赖治理方式。版本号只在"基线"一节维护。

## 一、基线（全新项目推荐起点）

本基线只作为全新项目的推荐起点。存量项目以实际探测版本为准，偏离项记录在 `openspec/project-profile.md`，不要求升级或重构。

| 组件 | 基线版本 | 说明 |
| --- | --- | --- |
| JDK | 17 | LTS，编译与运行一致 |
| Spring Boot | 3.3.x | 示例 3.3.5，按官方 release notes 选择补丁 |
| Spring Cloud | 2023.0.x | 与 Spring Boot 3.3 对齐 |
| Spring Cloud Alibaba | 2023.0.3.x | 示例 2023.0.3.2，与 Spring Cloud 2023.0.3 对齐 |
| MyBatis-Plus / MyBatis | 由项目选定一种 | 不混用两套 ORM 风格 |
| Testcontainers | 由 Spring Boot BOM 管理 | 集成测试使用 |

BOM 坐标：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>${spring-boot.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>${spring-cloud.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>${spring-cloud-alibaba.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

规则：

- 其他文档引用基线，不重复写版本号。
- 新项目按基线建立父 POM；存量项目沿用现有 BOM 与版本管理方式。
- 项目实际版本与基线不一致时，先说明差异，再按项目版本调整，不强行套用基线 API。
- 偏离项写入项目画像，升级作为独立决策。
- 升级时只改父 POM 的版本属性，并验证全模块编译、单元测试与集成测试。

## 一之二、存量项目评估

在存量项目中改动结构或依赖前：

- 先读 `openspec/project-profile.md`，核对模块划分、父 POM/BOM 与构建命令。
- 已有父 POM 或 BOM 时沿用，不引入第二个 BOM。
- 不为了符合推荐结构而重排模块或另建平行工程。
- 多模块仓库只评估本次涉及的模块，说明仓库级共享配置的影响。
- 模块结构与推荐不同时，按现有结构放置代码，差异记录在画像。

## 一之三、组件与版本决策记录

新增组件或升级版本前，依次评估：

1. 现有组件是否够用
2. 运维成本与团队熟悉度
3. 许可、安全与合规
4. 可观测性与故障处理
5. 迁移与退出成本

与用户讨论并达成一致后，把日期、背景、备选、结论、影响与批准人写入画像的"组件与版本决策记录"。存量项目优先复用已有注册中心、MQ 与缓存，避免同类中间件出现第二套。

## 二、模块划分

推荐结构：

```text
order-service/
├── order-api/             # 对外 DTO / VO / Feign 契约
├── order-common/          # 通用响应、异常、工具（无业务）
├── order-domain/          # 领域模型与领域服务
├── order-infrastructure/  # Mapper、外部客户端、消息
├── order-application/     # 应用服务、事务编排
└── order-boot/            # 启动模块：配置与装配
```

依赖规则：

- 单向依赖：boot → application → domain ← infrastructure；api 被 application 与外部消费者依赖。
- 启动模块只做装配，不承载业务逻辑。
- 禁止循环依赖；跨模块协作通过 api 契约或事件。
- common 保持无业务语义，避免变成"什么都往里塞"的模块。

## 三、父 POM

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <groupId>${groupId}</groupId>
    <artifactId>${artifactId}-parent</artifactId>
    <version>${revision}</version>
    <packaging>pom</packaging>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.3.5</spring-boot.version>
        <spring-cloud.version>2023.0.3</spring-cloud.version>
        <spring-cloud-alibaba.version>2023.0.3.2</spring-cloud-alibaba.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- 三个 BOM 见"基线"一节 -->
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-enforcer-plugin</artifactId>
                    <executions>
                        <execution>
                            <id>enforce</id>
                            <goals><goal>enforce</goal></goals>
                            <configuration>
                                <rules>
                                    <requireJavaVersion>
                                        <version>[17,)</version>
                                    </requireJavaVersion>
                                    <dependencyConvergence/>
                                </rules>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.jacoco</groupId>
                    <artifactId>jacoco-maven-plugin</artifactId>
                    <version>0.8.12</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

规则：

- 子模块不声明受管依赖版本。
- 版本属性集中在父 POM。
- 构建命令默认使用 `mvn verify`（含测试与覆盖率检查）。

## 四、依赖收敛

检查：

```bash
mvn -q dependency:tree -Dverbose
mvn -q enforcer:enforce
```

常见问题与处理：

| 问题 | 处理 |
| --- | --- |
| 同时出现 Logback 与 Log4j2 实现 | 排除其一，保留项目统一实现 |
| 多个版本的同一库 | 在父 POM 显式收敛版本 |
| 引入基线已移除的旧组件 | 替换为受管依赖 |
| 子模块自行覆写版本 | 删除版本声明，交由父 POM |
| 传递依赖带入不必要的 starter | 用 `<exclusions>` 排除 |

依赖收敛警告必须消除，不允许"带着警告构建"。

## 五、配置管理

`order-boot/src/main/resources/application.yml`

```yaml
server:
  port: ${SERVER_PORT:8080}
  shutdown: graceful

spring:
  application:
    name: order-service
  lifecycle:
    timeout-per-shutdown-phase: 30s
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_ADDR:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:public}
      config:
        server-addr: ${NACOS_ADDR:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:public}
        group: ${NACOS_GROUP:DEFAULT_GROUP}
  datasource:
    url: jdbc:mysql://${MYSQL_HOST:127.0.0.1}:${MYSQL_PORT:3306}/${MYSQL_DB:order}
    username: ${MYSQL_USER:order}
    password: ${MYSQL_PASSWORD}
  data:
    redis:
      host: ${REDIS_HOST:127.0.0.1}
      port: ${REDIS_PORT:6379}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

规则：

- 密钥、口令、令牌不提交仓库；使用环境变量或密钥管理。
- 提供本地可运行的默认值，或 README 明确列出所需中间件。
- 环境隔离使用 namespace / group / profile，不复制粘贴密钥。
- 明确哪些配置可热更新；连接池与线程池变更需重启。

## 六、可观测与运维配置

- 暴露 `health`、`metrics`、`prometheus`，管理端口不暴露到公网。
- 区分存活探针与就绪探针：就绪探针在依赖未就绪时应失败。
- 开启优雅停机：`server.shutdown=graceful` 与合理超时。
- 记录启动参数与服务版本，便于排障。

```yaml
management:
  endpoint:
    health:
      show-details: when_authorized
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

## 七、容器化与部署

`Dockerfile`

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn -q -DskipTests dependency:go-offline
COPY src ./src
RUN mvn -q -DskipTests package

FROM eclipse-temurin:17-jre
WORKDIR /app
RUN useradd -r -u 10001 appuser
COPY --from=build /build/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:InitialRAMPercentage=50.0", \
  "-Duser.timezone=Asia/Shanghai", \
  "-Dfile.encoding=UTF-8", \
  "-jar", "app.jar"]
```

运行参数建议：

- 使用容器感知的内存参数，避免被 OOMKill。
- 设置时区与编码，避免时间与中文乱码问题。
- 不在运行镜像中包含构建工具与源码。

Kubernetes 探针：

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

## 八、检查清单

- [ ] 基线版本只在本文件维护
- [ ] 子模块没有自行声明受管依赖版本
- [ ] `mvn enforcer:enforce` 无收敛告警
- [ ] 没有明文密钥入库
- [ ] 本地启动依赖的中间件已说明
- [ ] 健康检查区分存活与就绪
- [ ] 优雅停机配置生效
- [ ] 镜像为多阶段构建且以非 root 用户运行
