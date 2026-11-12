# 脚手架测试模板

与 `scaffolding-templates.md` 配套使用。行为代码先落测试，再落实现。

## 使用方式

- 先写测试并运行，确认失败原因是行为缺失（RED）。
- 再写最小实现，运行同一测试通过（GREEN）。
- 在回复中附 RED 与 GREEN 的命令与结果摘要。
- 只对本次改动涉及的层生成测试；简单改动不启动容器。

## 一、单元测试

`src/test/java/${basePackage}/application/${Entity}ServiceImplTest.java`

```java
package ${basePackage}.application;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ${Entity}ServiceImplTest {

    @Mock
    private ${Entity}Mapper ${entity}Mapper;

    @InjectMocks
    private ${Entity}ServiceImpl ${entity}Service;

    @Test
    void create_duplicateBizNo_throwsConflict() {
        // given
        when(${entity}Mapper.selectByBizNo("NO-1")).thenReturn(existing());

        // when / then
        assertThatThrownBy(() -> ${entity}Service.create(duplicateDto()))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining("重复");
    }

    @Test
    void create_validRequest_returnsVo() {
        // given
        when(${entity}Mapper.insert(any())).thenReturn(1);

        // when
        ${Entity}VO vo = ${entity}Service.create(validDto());

        // then
        assertThat(vo).isNotNull();
        assertThat(vo.getBizNo()).isEqualTo("NO-1");
    }
}
```

时间相关逻辑注入 `Clock`：

```java
@Test
void isTimeout_createdLongAgo_returnsTrue() {
    Clock fixed = Clock.fixed(Instant.parse("2026-01-01T10:00:00Z"), ZoneOffset.UTC);
    OrderTimeoutChecker checker = new OrderTimeoutChecker(fixed);

    assertThat(checker.isTimeout(orderCreatedAt("2026-01-01T09:00:00Z"))).isTrue();
}
```

命令：

```bash
mvn -q -Dtest=${Entity}ServiceImplTest test
```

### JUnit 4 存量项目

项目使用 JUnit 4 时沿用现有注解与运行器，不引入 JUnit 5：

```java
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.MockitoJUnitRunner;

@RunWith(MockitoJUnitRunner.class)
public class ${Entity}ServiceImplTest {

    @Mock
    private ${Entity}Mapper ${entity}Mapper;

    @InjectMocks
    private ${Entity}ServiceImpl ${entity}Service;

    @Test
    public void create_duplicateBizNo_throwsConflict() {
        // given / when / then
    }
}
```

## 二、MockMvc 契约测试

`src/test/java/${basePackage}/api/${Entity}ControllerIT.java`

```java
package ${basePackage}.api;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
class ${Entity}ControllerIT {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void create_invalidBody_returnsParamInvalid() throws Exception {
        mockMvc.perform(post("/api/v1/${entityPath}")
                        .contentType("application/json")
                        .content("{}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value("A0400"));
    }

    @Test
    void create_validRequest_returnsOk() throws Exception {
        mockMvc.perform(post("/api/v1/${entityPath}")
                        .contentType("application/json")
                        .content("""
                                {"bizNo":"NO-1","amount":100}
                                """))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value("0"))
                .andExpect(jsonPath("$.data.bizNo").value("NO-1"));
    }
}
```

## 三、Testcontainers 集成测试

`src/test/java/${basePackage}/infrastructure/${Entity}MapperIT.java`

```java
package ${basePackage}.infrastructure;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
class ${Entity}MapperIT {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("${dbName}")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Test
    void selectPage_returnsFilteredRows() {
        // given：插入不同状态数据
        // when：按状态查询
        // then：只返回匹配行，分页正确
    }
}
```

要求：

- 测试数据自带准备与清理，用例互不影响。
- 不使用固定主键依赖执行顺序。
- 并行执行时使用唯一业务键前缀。
- 无 Docker 时使用条件跳过，并在结果中说明。

命令：

```bash
mvn -q -Dtest=${Entity}MapperIT test
```

## 四、消息消费测试要点

- 重复消息：同一消息键投递两次，副作用只发生一次。
- 消费失败：按配置重试，超过次数进入死信。
- 顺序性：同一消息键的消息按发送顺序处理，或说明为何不要求顺序。
- traceId：消费端日志携带消息头中的 traceId。

## 五、依赖坐标

测试模板假设工程已引入：

- `spring-boot-starter-test`（JUnit 5、Mockito、AssertJ、MockMvc）
- `org.testcontainers:junit-jupiter`
- `org.testcontainers:mysql`
- 按需 `org.testcontainers:rocketmq`、`org.testcontainers:redis`（或对应镜像容器）

缺失时在回复中列出坐标，不输出无法编译的测试。

## 六、测试确认清单

- [ ] 行为代码先有失败测试（RED）
- [ ] RED 失败原因是行为缺失
- [ ] 实现后同一测试通过（GREEN）
- [ ] 外部依赖已替身化，时间通过 `Clock` 注入
- [ ] 只对本次改动涉及的层生成测试
- [ ] 集成测试数据可重复、可并行
- [ ] 无容器环境时已标注跳过条件
- [ ] 命令与结果已附在完成说明中
