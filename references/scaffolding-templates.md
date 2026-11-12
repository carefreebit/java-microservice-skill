# 脚手架模板

模板用于生成与项目约定一致的分层代码。所有模板以目标文件路径开头，使用 `${...}` 占位符；生成前先确认清单。测试模板见 `scaffolding-test-templates.md`。

## 使用说明

- 生成前先读 `openspec/project-profile.md`；存量项目按画像中的包结构、基类、响应、异常、转换器、ORM 与测试框架生成。
- 项目已有统一响应、业务异常、错误码、基类或转换器时直接复用，不生成平行实现。
- 行为代码（Service / 领域逻辑 / API 契约）先给测试，再给实现，并注明 RED/GREEN 验证命令。
- DAO、DTO、配置等装配代码可与测试同步产出。
- 占位符必须替换为项目实际值；无法确定的先列确认清单，不臆造表结构。
- 包名以 `${basePackage}` 表示，例如 `com.example.order`。

## 一、包结构

```text
src/main/java/${basePackage}/
├── api/                  # Controller、DTO、VO
├── application/          # 应用服务、事务编排
├── domain/               # 领域模型、领域服务
├── infrastructure/       # Mapper、外部客户端、消息
└── common/               # 统一响应、异常、工具
```

依赖方向：api → application → domain ← infrastructure。禁止反向依赖。

## 一之二、存量项目适配

生成前检查并按现有设施复用：包结构放入现有路径；统一响应引用 `Result<T>` / `ApiResponse` / `R`；业务异常与错误码沿用现有类型；基类继承 `BaseController` / `BaseService`；转换器、ORM、测试框架沿用项目写法。已有 `Result<T>` 与 `BizException` 时不再生成 `ApiResponse` 与 `BusinessException`。

版本差异：Boot 2 用 `javax.validation` / `javax.servlet`，Boot 3 用 `jakarta`；Java 8 不生成 record、var、text block；Swagger 2 沿用 `io.swagger.annotations`；无法判断时先提问。

```java
// Boot 2 / JDK 8 存量项目的校验导入
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
```

已有 `Result<T>` / `BizException` 时直接复用，不再生成下文中的 `ApiResponse` 与 `BusinessException`。

## 二、DTO / VO / DO / Converter

`api/dto/${Entity}CreateDTO.java`

```java
package ${basePackage}.api.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import lombok.Data;

@Data
public class ${Entity}CreateDTO {
    @NotBlank(message = "业务单号不能为空")
    private String bizNo;

    @NotNull(message = "金额不能为空")
    @Positive(message = "金额必须大于 0")
    private Long amount;

    private String remark;
}
```

查询入参 `${Entity}QueryDTO` 复用同一模式：`keyword` 可选，`pageNo` 从 1 开始，`pageSize` 默认 20 且上限 200，均用 `@Min/@Max` 校验。

`api/vo/${Entity}VO.java`

```java
package ${basePackage}.api.vo;

import lombok.Data;

@Data
public class ${Entity}VO {
    private Long id;
    private String bizNo;
    private Integer status;
    private Long amount;
}
```

`infrastructure/persistence/${Entity}DO.java`

```java
package ${basePackage}.infrastructure.persistence;

import lombok.Data;

@Data
public class ${Entity}DO {
    private Long id;
    private String bizNo;
    private Integer status;
    private Long amount;
    private Integer version;
    private Long createdAt;
    private Long updatedAt;
}
```

`api/converter/${Entity}Converter.java`

```java
package ${basePackage}.api.converter;

import org.mapstruct.Mapper;
import org.mapstruct.MappingConstants;

@Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
public interface ${Entity}Converter {
    ${Entity}VO toVo(${Entity}DO entity);
    ${Entity}DO toDo(${Entity}CreateDTO dto);
}
```

## 三、Controller

`api/${Entity}Controller.java`

```java
package ${basePackage}.api;

import ${basePackage}.common.ApiResponse;
import ${basePackage}.api.dto.${Entity}CreateDTO;
import ${basePackage}.api.dto.${Entity}QueryDTO;
import ${basePackage}.api.vo.${Entity}VO;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@Tag(name = "${Entity} 管理")
@RestController
@RequestMapping("/api/v1/${entityPath}")
@RequiredArgsConstructor
public class ${Entity}Controller {

    private final ${Entity}Service ${entity}Service;

    @Operation(summary = "创建${Entity}")
    @PostMapping
    public ApiResponse<${Entity}VO> create(@Valid @RequestBody ${Entity}CreateDTO dto) {
        return ApiResponse.ok(${entity}Service.create(dto));
    }

    @Operation(summary = "分页查询${Entity}")
    @GetMapping
    public ApiResponse<PageResult<${Entity}VO>> page(@Valid ${Entity}QueryDTO dto) {
        return ApiResponse.ok(${entity}Service.page(dto));
    }
}
```

## 四、Service

`application/${Entity}Service.java`

```java
package ${basePackage}.application;

import ${basePackage}.api.dto.${Entity}CreateDTO;
import ${basePackage}.api.dto.${Entity}QueryDTO;
import ${basePackage}.api.vo.${Entity}VO;
import ${basePackage}.common.PageResult;

public interface ${Entity}Service {
    ${Entity}VO create(${Entity}CreateDTO dto);
    PageResult<${Entity}VO> page(${Entity}QueryDTO dto);
}
```

`application/${Entity}ServiceImpl.java`

```java
package ${basePackage}.application;

import ${basePackage}.api.converter.${Entity}Converter;
import ${basePackage}.infrastructure.persistence.${Entity}Mapper;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class ${Entity}ServiceImpl implements ${Entity}Service {

    private final ${Entity}Mapper ${entity}Mapper;
    private final ${Entity}Converter converter;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public ${Entity}VO create(${Entity}CreateDTO dto) {
        // 1. 业务校验
        // 2. 幂等检查（业务唯一键 / 去重表）
        // 3. 组装 DO 并写入
        // 4. 返回 VO
        return null;
    }
}
```

## 五、Mapper

`infrastructure/persistence/${Entity}Mapper.java`

```java
package ${basePackage}.infrastructure.persistence;

import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

import java.util.List;

@Mapper
public interface ${Entity}Mapper {

    int insert(${Entity}DO entity);

    ${Entity}DO selectById(@Param("id") Long id);

    List<${Entity}DO> selectPage(@Param("keyword") String keyword, @Param("offset") int offset, @Param("limit") int limit);
}
```

## 六、统一响应与错误码

`common/ApiResponse.java`

```java
package ${basePackage}.common;

import lombok.Data;

@Data
public class ApiResponse<T> {

    private String code;
    private String message;
    private T data;

    public static <T> ApiResponse<T> ok(T data) {
        return build(ErrorCode.SUCCESS, data);
    }

    public static <T> ApiResponse<T> fail(ErrorCode errorCode) {
        return build(errorCode, null);
    }

    private static <T> ApiResponse<T> build(ErrorCode errorCode, T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode(errorCode.getCode());
        response.setMessage(errorCode.getMessage());
        response.setData(data);
        return response;
    }
}
```

`common/ErrorCode.java`

```java
package ${basePackage}.common;

import lombok.Getter;

@Getter
public enum ErrorCode {

    SUCCESS("0", "成功"),
    PARAM_INVALID("A0400", "参数校验失败"),
    NOT_FOUND("A0404", "资源不存在"),
    CONFLICT("A0409", "并发冲突"),
    SYSTEM_ERROR("B0001", "系统繁忙，请稍后重试");

    private final String code;
    private final String message;

    ErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }
}
```

业务错误码按模块分段集中定义，禁止在业务代码中写字面量。

## 七、全局异常处理

`common/BusinessException` 继承 `RuntimeException`，持有 `ErrorCode` 与可选自定义消息；业务代码只抛这一种受控异常。

`common/GlobalExceptionHandler.java`

```java
package ${basePackage}.common;

import jakarta.validation.ConstraintViolationException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public ApiResponse<Void> handleBusiness(BusinessException e) {
        log.warn("业务异常, code={}, message={}", e.getCode(), e.getMessage());
        return ApiResponse.fail(e.getErrorCode());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<Void> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(error -> error.getField() + ": " + error.getDefaultMessage())
                .findFirst().orElse("参数校验失败");
        return ApiResponse.fail(ErrorCode.PARAM_INVALID, message);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public ApiResponse<Void> handleConstraint(ConstraintViolationException e) {
        return ApiResponse.fail(ErrorCode.PARAM_INVALID, e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public ApiResponse<Void> handleUnknown(Exception e) {
        log.error("未处理异常", e);
        return ApiResponse.fail(ErrorCode.SYSTEM_ERROR);
    }
}
```

注意：未知异常不向前端泄漏堆栈；日志必须带 traceId。

## 八、参数校验分组

新增与修改共用 DTO 时用校验分组（`CreateGroup` / `UpdateGroup`）区分规则，Controller 使用 `@Validated(CreateGroup.class)`，不在业务代码中写分支校验。

## 九、日志与 traceId

`common/TraceIdFilter.java`

```java
package ${basePackage}.common;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.UUID;

@Component
public class TraceIdFilter implements Filter {

    private static final String TRACE_ID = "traceId";

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String traceId = ((HttpServletRequest) request).getHeader(TRACE_ID);
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }
        MDC.put(TRACE_ID, traceId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(TRACE_ID);
        }
    }
}
```

异步传递使用 `TaskDecorator`：任务提交时复制 MDC，执行结束后清理；消息消费端从消息头读取 traceId 并写入 MDC。

消息消费端从消息头读取 traceId 并写入 MDC；消息发送端把当前 traceId 写入消息头。

## 十、OpenAPI 注解

在 Controller 方法上使用 `@Operation(summary=..., description=...)` 与 `@ApiResponses` 声明成功、参数错误、重复请求等响应，注解内容必须与实际行为一致；废弃接口标注 `deprecated = true` 并说明替代接口。

## 十一、使用前确认清单

- [ ] `${basePackage}`、`${entity}`、`${Entity}`、`${table}` 已替换为实际值
- [ ] 表结构、字段类型与 DO 一致
- [ ] 业务错误码已分配到对应模块段
- [ ] 事务边界位于 Service 实现，Controller 无事务
- [ ] 依赖坐标已存在：validation、mybatis、mapstruct、junit5、testcontainers
- [ ] 行为代码已有失败测试（RED）
- [ ] 集成测试所需容器环境可用，或已标注跳过条件
