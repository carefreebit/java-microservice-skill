# 测试规范

覆盖单元测试、集成测试与测试先行（TDD）的交付要求。测试不是收尾动作，而是实现的一部分。

## 一、测试层次与覆盖范围

按本次改动涉及的层选择测试，不做无意义的全量生成：

| 改动层 | 必测内容 | 方式 |
| --- | --- | --- |
| Service / 领域逻辑 | 正常、边界、异常分支 | JUnit 5 + Mockito 单元测试 |
| 对外 API 契约 | 成功、参数校验失败、权限/不存在 | MockMvc 集成测试 |
| DAO / SQL / 索引 | 查询条件、分页、映射、唯一约束 | `@SpringBootTest` + Testcontainers |
| 消息消费者 | 正常消费、重复消息幂等、重试与死信 | Testcontainers RocketMQ 或集成环境 |
| 缓存 / 分布式锁 | 命中、过期、并发安全 | Testcontainers Redis |

只改业务分支时不启动容器；只改 SQL 时不重复写 Service 单元测试。

## 二、单元测试规范

- 新项目使用 JUnit 5 + Mockito，断言使用 AssertJ；存量项目沿用 JUnit 4 或现有测试框架，不引入第二套。
- 遵循 AIR：自动化（Automatic）、独立（Independent）、可重复（Repeatable）。
- 每个用例聚焦一个行为，断言结果值与副作用。
- 外部依赖（HTTP 客户端、消息、缓存）全部替身化。
- 时间通过注入的 `Clock` 获取，禁止依赖系统时钟。
- 命名：`方法名_场景_预期`，或使用中文 `@DisplayName`。

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceImplTest {

    @Mock
    private StockClient stockClient;

    @Mock
    private OrderMapper orderMapper;

    @InjectMocks
    private OrderServiceImpl orderService;

    @Test
    void cancel_paidOrder_throwsBusinessException() {
        OrderDO order = OrderFixture.order(OrderStatus.PAID);
        when(orderMapper.selectById(1L)).thenReturn(order);

        assertThatThrownBy(() -> orderService.cancel(1L))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining("当前状态不可取消");
    }
}
```

时间注入：

```java
@Service
public class OrderTimeoutChecker {
    private final Clock clock;

    public OrderTimeoutChecker(Clock clock) {
        this.clock = clock;
    }

    public boolean isTimeout(OrderDO order) {
        return order.getCreatedAt().plusMinutes(30).isBefore(clock.instant());
    }
}
```

测试中注入 `Clock.fixed(...)`，不写 `Thread.sleep`。

## 三、集成测试规范

- 使用 `@SpringBootTest` 加载真实应用上下文。
- 新项目使用 Testcontainers 提供 MySQL、Redis、RocketMQ；存量项目沿用现有集成测试设施（共享测试库、本地中间件或嵌入式数据库），并在画像与报告中说明。
- API 使用 `MockMvc` 或 `WebTestClient` 验证契约与错误码。
- 不使用 H2 替代 MySQL 作为默认方案；H2 仅用于无 Docker 的降级验证。
- 测试数据自带准备与清理，用例互不影响，可并行执行。

DAO 示例：

```java
@SpringBootTest
@Testcontainers
class OrderMapperIT {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    private OrderMapper orderMapper;

    @Test
    void selectPage_withStatus_filtersRows() {
        // given：插入 3 条不同状态数据
        // when：按状态查询
        // then：只返回匹配状态的行
    }
}
```

消息消费测试关注：

- 正常消息被消费一次。
- 重复消息不产生重复副作用。
- 消费失败按策略重试。
- 超过重试次数进入死信。

数据隔离：

- 使用独立测试库或每次测试清理表。
- 不使用固定主键依赖执行顺序。
- 并行执行时使用唯一业务键前缀。

## 三之二、存量改造测试

### 最小测试地基

项目没有可运行的测试时，先补：

1. 与项目版本兼容的测试依赖
2. 一个最小测试基类或配置
3. 一个冒烟测试，确认能跑

不允许以"项目原本没有测试"为由跳过验证。

### 特征测试

特征测试锁定现状，不表达期望：

1. 选出将被触碰的行为
2. 写测试覆盖现有输入与输出
3. 运行得到 BASELINE 全绿
4. 修改代码
5. 同一批测试仍全绿（REGRESSION）

只覆盖本次触碰的行为，不补全模块。测试可达性允许最小改造（抽接口、子类覆盖、包装外部调用），但要在任务中记录。

### 证据格式

```text
BASELINE
命令：mvn -q -Dtest=OrderAmountCalculatorTest test
结果：Tests run: 6, Failures: 0, Errors: 0

REGRESSION
命令：mvn -q -Dtest=OrderAmountCalculatorTest test
结果：Tests run: 6, Failures: 0, Errors: 0

RED
命令：mvn -q -Dtest=OrderAmountCalculatorTest#newRule test
结果：FAILED — 期望新规则，实际仍为旧规则

GREEN
命令：mvn -q -Dtest=OrderAmountCalculatorTest test
结果：Tests run: 8, Failures: 0, Errors: 0
```

纯重构只需要 BASELINE 与 REGRESSION；行为变化才需要 RED 与 GREEN。

## 四、测试先行（TDD）

### 适用范围

行为代码必须测试先行：

- Service / 领域逻辑
- 校验规则与状态流转
- API 契约与响应结构

装配代码可同步或后置，见"豁免清单"。

### RED-GREEN-REFACTOR

1. **RED**：写一个测试，表达期望行为；运行，确认失败原因是行为缺失，而不是编译错误、依赖缺失或环境问题。
2. **GREEN**：写最小实现让该测试通过；运行相关测试，确认无回归。
3. **REFACTOR**：在不改变行为的前提下重构；再次运行测试。

禁止顺序：先写实现，再补一组"刚好通过"的测试。

### 完整示例

需求：订单取消接口，只有待支付状态可取消。

第一步，写测试：

```java
@Test
void cancel_paidOrder_throwsBusinessException() {
    when(orderMapper.selectById(1L)).thenReturn(orderWithStatus(PAID));

    assertThatThrownBy(() -> orderService.cancel(1L))
            .isInstanceOf(BusinessException.class);
}
```

运行：

```bash
mvn -q -Dtest=OrderServiceImplTest#cancel_paidOrder_throwsBusinessException test
```

预期 RED：测试失败，提示没有抛出 `BusinessException`（方法可能尚不存在或未校验状态）。

第二步，写最小实现：

```java
@Override
@Transactional(rollbackFor = Exception.class)
public void cancel(Long orderId) {
    OrderDO order = orderMapper.selectById(orderId);
    if (!OrderStatus.WAIT_PAY.equals(order.getStatus())) {
        throw new BusinessException(OrderErrorCode.STATUS_ILLEGAL);
    }
    orderMapper.updateStatus(orderId, OrderStatus.CANCELLED);
}
```

第三步，运行：

```bash
mvn -q -Dtest=OrderServiceImplTest test
```

预期 GREEN：该测试通过，且同类其他测试无回归。

第四步，补充正常路径、订单不存在、库存回补失败等用例，再重构。

## 五、RED / GREEN 证据格式

每次行为任务完成时，在回复中给出：

```text
RED
命令：mvn -q -Dtest=OrderServiceImplTest#cancel_paidOrder_throwsBusinessException test
结果：FAILED — 预期抛出 BusinessException，实际未抛出（行为缺失）

GREEN
命令：mvn -q -Dtest=OrderServiceImplTest test
结果：Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
```

要求：

- RED 的失败原因必须指向业务行为，不能是编译错误、网络错误或容器未启动。
- GREEN 必须包含相关测试集的通过数量。
- 没有运行输出，不得以"已写测试"作为完成依据。

## 六、TDD 豁免清单

以下场景可以不测试先行，但必须在任务中写明理由与替代验证：

| 场景 | 替代验证 |
| --- | --- |
| DAO / SQL / 索引调整 | Testcontainers 集成测试 |
| 脚手架、DTO、Converter、配置 | 编译 + 契约测试或启动验证 |
| 分布式事务、消息最终一致性 | 集成测试、故障注入、对账脚本验证 |
| 探索性原型 | 明确标注原型，不进入正式完成标准 |

豁免不得用于规避行为测试。明显改变业务规则的任务不能标为豁免。

## 七、覆盖率与门禁

- 新增或修改的业务代码行覆盖率默认不低于 60%。
- 核心业务分支（支付、取消、退款、库存扣减等）必须全覆盖。
- 覆盖率阈值集中在构建配置中，可按项目调整。

JaCoCo 思路：

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.60</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

门禁：

- `mvn verify` 通过前不得勾选任务。
- 行为任务缺少 RED 证据不得勾选。
- 失败用例不得注释、跳过或放宽断言来"通过"。

## 八、测试与任务同组交付

- change 的 `tasks.md` 中，测试任务必须与对应实现位于同一任务组。
- 实现任务只有在测试编写并通过后才可勾选。
- 禁止把所有测试集中到最后统一补写。

示例任务结构：

```text
## 2. 订单取消
- [ ] 2.1 先写取消状态校验的失败测试并运行（RED）
- [ ] 2.2 实现取消逻辑并运行测试通过（GREEN）
- [ ] 2.3 补充库存回补失败的单元测试
- [ ] 2.4 补充取消接口的 MockMvc 契约测试
```

## 九、结果报告与失败处理

完成一组任务时报告：

- 执行命令
- 通过 / 失败 / 跳过数量
- 覆盖率数字
- 未执行项及原因

失败处理：

- 原样报告失败输出，不截断关键堆栈。
- 修复后重跑，不通过不得宣告完成。
- 不允许通过删除用例、降低断言强度、跳过测试来伪造通过。

## 十、无 Docker 环境

Testcontainers 无法启动时：

- 明确报告哪些集成测试被跳过。
- 说明采用的替代验证（单元测试、契约测试、手工验证）。
- 不得声称"全部测试通过"。
- 可在测试上使用 `@EnabledIfEnvironmentVariable` 或 `assumeTrue` 条件跳过，并在构建摘要中列出。

## 十一、检查清单

- [ ] 行为代码先有失败测试（RED）
- [ ] RED 失败原因是行为缺失
- [ ] 实现后同一测试通过（GREEN）
- [ ] 相关测试集无回归
- [ ] 外部依赖已替身化，时间通过 Clock 注入
- [ ] 改动的 DAO / API 有对应集成测试
- [ ] 测试数据可重复，用例互不影响
- [ ] 覆盖率达标
- [ ] 测试任务与实现同组
- [ ] 失败与跳过项如实报告
