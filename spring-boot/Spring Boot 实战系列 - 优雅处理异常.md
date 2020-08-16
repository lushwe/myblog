# Spring Boot 实战系列 - 优雅处理异常

## 准备工作
- `pom.xml` 依赖如下
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernate.validator</groupId>
        <artifactId>hibernate-validator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

- 开发一个 `Controller` ，代码如下
```java
@Validated
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public String getUserById(@Min(value = 0, message = "参数不合法") @PathVariable("id") Long id) {
        return "zhangsan";
    }
}
```

## 异常处理

- 方法一
```java
@Component
public class ApiHandlerExceptionResolver extends SimpleMappingExceptionResolver {

    @Override
    protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object o, Exception e) {

        response.setCharacterEncoding("UTF-8");

        ApiResultEntity entity = ExceptionUtils.buildApiResultEntity(e);

        PrintWriter writer = null;
        try {
            writer = response.getWriter();
            writer.write(JSON.toJSONString(entity));
            writer.flush();
        } catch (IOException ioe) {
            ioe.printStackTrace();
        } finally {
            if (writer != null) {
                writer.close();
            }
        }
        return new ModelAndView();
    }
}
```

- 方法二
```java
@Slf4j
@ControllerAdvice
public class ApiResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler
    protected ResponseEntity<Object> doHandleApiException(Exception e, WebRequest request) {

        ApiResultEntity entity = ExceptionUtils.buildApiResultEntity(e);

        return handleExceptionInternal(e, entity, new HttpHeaders(), HttpStatus.OK, request);
    }
}
```

- 相关类代码如下
```java
public class ExceptionUtils {

    public static ApiResultEntity buildApiResultEntity(Exception e) {
        ApiResultEntity entity = new ApiResultEntity();
        if (e instanceof IllegalArgumentException) {
            entity.setCode(HttpStatus.CONFLICT.value());
            entity.setMsg(e.getMessage());
        } else if (e instanceof ConstraintViolationException) {
            entity.setCode(HttpStatus.CONFLICT.value());
            entity.setMsg(e.getMessage());
        } else {
            entity.setCode(HttpStatus.INTERNAL_SERVER_ERROR.value());
            entity.setMsg(HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase());
        }
        return entity;
    }
}

@Data
public class ApiResultEntity<T> {

    private int code;
    private String msg;
    private T data;
}
```

- 总结：方法一相对复杂，推荐使用方法二

---

本文完。
