# 阿里巴巴 Java 编码规约

本文档摘取《阿里巴巴 Java 开发手册》中与后端微服务开发相关的条款，并给出可判定的整改口径。

## 使用说明

- **【强制】**：必须遵守，违反按缺陷处理。
- **【推荐】**：默认遵守，偏离需说明理由。
- **【参考】**：按场景选择。

评审输出必须写成"类别【级别】：问题 → 建议"，例如：

```text
并发规约【强制】：使用 Executors.newFixedThreadPool 创建线程池。
建议：改用 ThreadPoolExecutor，指定核心数、最大数、有界队列、线程命名与拒绝策略。
```

## 一、命名规约

### 1.1 基本命名

- 【强制】类名 UpperCamelCase；抽象类以 `Abstract` 或 `Base` 开头；异常类以 `Exception` 结尾；测试类以 `Test` 结尾。
- 【强制】方法与变量 lowerCamelCase；常量 UPPER_SNAKE_CASE；包名全小写。
- 【强制】禁止拼音与无意义缩写，使用业务语义明确的英文。
- 【强制】POJO 布尔字段不加 `is` 前缀，避免序列化与框架映射歧义。

```java
// 反例
private Boolean isDeleted;
private int ddzt;
public List<Order> getDingdanList() { ... }

// 正例
private Boolean deleted;
private OrderStatus status;
public List<Order> listOrders() { ... }
```

### 1.2 分层对象命名

- 【强制】数据库映射对象使用 `DO`，接口入参使用 `DTO`，出参使用 `VO`，领域对象使用 `BO`。
- 【强制】禁止用 DO 直接作为接口出入参，避免暴露存储结构。
- 【推荐】查询参数单独使用 `XxxQueryDTO`，避免复用写模型。

```java
OrderDO        // 持久化
OrderQueryDTO  // 查询入参
OrderCreateDTO // 创建入参
OrderVO        // 接口出参
```

## 二、常量与魔法值

- 【强制】禁止未经解释的魔法值；`-1`、`0`、`1` 作为约定值除外。
- 【强制】状态、类型、错误码使用枚举或集中常量，不在业务代码中散落字面量。
- 【推荐】常量按业务域归类，禁止一个巨大的 `Constants` 类。

```java
// 反例
if (order.getStatus() == 3) { ... }

// 正例
if (OrderStatus.COMPLETED.equals(order.getStatus())) { ... }
```

## 三、集合与并发安全

### 3.1 集合

- 【推荐】已知容量时指定初始容量，避免扩容：`new HashMap<>((int) (size / 0.75f) + 1)`。
- 【强制】`Arrays.asList` 返回定长列表，需要增删时包装为 `new ArrayList<>(...)`。
- 【强制】禁止在 `foreach` 中增删集合元素，使用 `Iterator` 或 `removeIf`。
- 【推荐】集合转数组使用 `toArray(new T[0])`。
- 【强制】多线程共享集合使用并发容器，不用 `HashMap` / `ArrayList`。

### 3.2 线程池

- 【强制】禁止使用 `Executors` 工厂方法创建线程池。
- 【强制】线程池必须显式设置核心数、最大数、有界队列、线程命名与拒绝策略。
- 【推荐】不同业务使用独立线程池，避免互相拖垮。

```java
// 反例
ExecutorService pool = Executors.newFixedThreadPool(10);

// 正例
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        8, 16, 60L, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(500),
        new ThreadFactoryBuilder().setNameFormat("order-notify-%d").build(),
        new ThreadPoolExecutor.CallerRunsPolicy());
```

### 3.3 时间与线程上下文

- 【强制】`SimpleDateFormat` 不是线程安全的，禁止作为共享静态变量；使用 `DateTimeFormatter`。
- 【强制】`ThreadLocal` 必须在 `finally` 中 `remove()`。
- 【推荐】异步线程与消息消费显式传递 traceId 与用户上下文。

```java
try {
    UserContext.set(user);
    handle();
} finally {
    UserContext.clear();
}
```

### 3.4 锁

- 【强制】加锁顺序一致，避免死锁。
- 【强制】锁内禁止执行远程调用、IO 或长时间计算。
- 【推荐】优先使用并发工具与无锁结构，避免手写 `synchronized` 大块逻辑。

## 四、异常处理

- 【强制】禁止空 `catch`，禁止捕获后吞掉。
- 【强制】业务异常使用统一异常体系并携带错误码；系统异常记录完整堆栈。
- 【强制】禁止捕获 `Throwable`；不要用异常控制正常流程。
- 【强制】事务方法声明 `@Transactional(rollbackFor = Exception.class)`。
- 【推荐】资源关闭使用 try-with-resources。

```java
// 反例
try {
    stockClient.deduct(id);
} catch (Exception e) {
}

// 正例
try {
    stockClient.deduct(id);
} catch (RemoteException e) {
    log.error("扣减库存失败, orderId={}", orderId, e);
    throw new BusinessException(OrderErrorCode.STOCK_DEDUCT_FAILED, e);
}
```

## 五、日志规约

- 【强制】使用 SLF4J 占位符，禁止字符串拼接。
- 【强制】异常日志保留异常对象，作为最后一个参数传入。
- 【强制】禁止输出密码、令牌、身份证、银行卡、手机号等敏感信息。
- 【推荐】日志包含关键业务键（订单号、用户 ID、traceId），便于检索。
- 【推荐】避免在循环中打印大日志；批量操作只记录汇总与失败样本。

```java
// 反例
log.info("取消订单 " + orderId + " 失败: " + e.getMessage());

// 正例
log.error("取消订单失败, orderId={}", orderId, e);
```

## 六、单元测试

- 【强制】核心业务分支必须有单元测试，遵循 AIR（自动化、独立、可重复）。
- 【强制】外部依赖通过 mock 或测试替身隔离，禁止依赖真实网络与数据库。
- 【强制】时间通过注入的 `Clock` 获取，禁止依赖系统时间。
- 【推荐】测试方法命名 `方法名_场景_预期`，或使用 `@DisplayName`。
- 【推荐】每个用例聚焦一个行为，断言结果值与副作用。

```java
@Test
void cancel_paidOrder_throwsBusinessException() {
    // given
    Order order = OrderFixture.paidOrder();
    // when / then
    assertThatThrownBy(() -> orderService.cancel(order.getId()))
            .isInstanceOf(BusinessException.class)
            .hasFieldOrPropertyWithValue("code", "ORDER_STATUS_ILLEGAL");
}
```

## 七、MySQL 访问规约

- 【强制】表名、字段名小写下划线；必须有主键。
- 【强制】禁止 `select *`，显式列出所需列。
- 【强制】禁止大事务；禁止在循环内逐条写库，使用批量插入并分批提交。
- 【强制】模糊查询禁止以 `%` 开头；索引列上避免函数与隐式类型转换。
- 【强制】更新必须带明确条件；推荐使用版本号做乐观锁。
- 【推荐】唯一索引用于幂等去重；分页查询使用覆盖索引。

```sql
-- 反例
select * from t_order where date(create_time) = '2026-01-01';

-- 正例
select id, order_no, status, amount
from t_order
where create_time >= '2026-01-01 00:00:00'
  and create_time <  '2026-01-02 00:00:00'
  and status = 10;
```

## 八、设计规约

- 【强制】Controller 只做参数校验与编排；业务规则在 Service；数据访问收敛在 Mapper / Repository。
- 【强制】事务边界位于 Service 实现层，Controller 不加事务注解。
- 【强制】禁止模块间循环依赖；跨模块通过公共 API 或事件协作。
- 【强制】跨服务交互通过接口契约，不共享数据库。
- 【推荐】复杂条件分支使用策略模式或枚举分派，避免超长 if-else。
- 【推荐】接口保持向后兼容，新增字段不删除既有字段。

```java
// 反例：胖 Controller
@PostMapping("/orders")
@Transactional
public Result create(@RequestBody OrderRequest req) {
    // 校验、计算、扣库存、写订单全在这里
}

// 正例：薄 Controller
@PostMapping("/orders")
public ApiResponse<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) {
    return ApiResponse.ok(orderService.create(dto));
}
```

## 九、快速判定表

| 现象 | 判定 | 整改方向 |
| --- | --- | --- |
| `Executors.newXxx` | 【强制】违规 | 改为 `ThreadPoolExecutor` |
| 空 catch | 【强制】违规 | 记录、包装或抛出 |
| 日志字符串拼接 | 【强制】违规 | SLF4J 占位符 |
| `select *` | 【强制】违规 | 显式列 |
| DO 作为接口出参 | 【强制】违规 | 转为 VO |
| 拼音命名 | 【强制】违规 | 业务英文命名 |
| `if (status == 3)` | 【强制】违规 | 枚举或常量 |
| 循环内单条 insert | 【强制】违规 | 批量写入 |
| Controller 含事务 | 【强制】违规 | 下沉到 Service |
| 无超时的远程调用 | 【强制】违规 | 设置连接与读取超时 |
