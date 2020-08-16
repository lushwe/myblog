# Spring Boot 实战系列 - 方法异步调用

> 一般在一个方法中需要处理多个任务，其中某些任务无关紧要（如发送短信、记录操作日志等），可以使用异步处理那些无关紧要的任务，从而提高整个请求的相应时间。下面演示使用 Spring Boot 快速开发方法异步处理

## `pom.xml` 依赖配置如下
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

## `Controller` 层代码
- `UserController` 代码如下
```java
@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/users/{id}")
    public UserDTO get(@PathVariable("id") Long id) {
        return userService.getById(id);
    }
}
```

## `Service` 层代码
- `UserService` 接口定义如下
```java
// 接口定义
public interface UserService {

    UserDTO getById(Long id);
}
```

- `UserServiceImpl` 实现 `UserService` 接口，代码如下

```java
// 接口实现
@Slf4j
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private LogService logService;

    @Override
    public UserDTO getById(Long id) {

        log.info("查询用户");
        UserDTO userDTO = new UserDTO();
        userDTO.setId(id);

        log.info("记录操作日志");
        logService.save();

        return userDTO;
    }
}
```

- `LogService` 接口定义如下
```java
// 接口定义
public interface LogService {

    void save();
}
```


- `LogServiceImpl` 实现 `LogService` 接口，代码如下
```java
// 接口实现
@Slf4j
@Service
public class LogServiceImpl implements LogService {

    @Async("logServiceExecutor") // @Async 表示该方法异步执行，"logServiceExecutor"用来指定线程池实例
    @Override
    public void save() {

        log.info("{},{},记录日志, start", Integer.toHexString(Thread.currentThread().hashCode()), Thread.currentThread().getName());
        try {
            // 休眠3秒，异步效果更明显
            Thread.sleep(3000);
        } catch (Exception e) {
            e.printStackTrace();
        }
        log.info("{},{},记录日志, end.", Integer.toHexString(Thread.currentThread().hashCode()), Thread.currentThread().getName());
    }
}
```

## 配置层
- 线程池配置代码如下
```java
@Slf4j
@EnableAsync
@Configuration
public class ExecutorConfig {

    @Bean("logServiceExecutor")
    public Executor logServiceExecutor() {

        log.info("启动 Log Service Executor");

        ThreadPoolExecutor executor = new ThreadPoolExecutor(3, 10,
                5L, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(512),
                new CustomizableThreadFactory("save-log-thread-"),
                new ThreadPoolExecutor.DiscardPolicy());

        return executor;
    }
}
```

## 总结
- 从代码可以看出，一共增加了两部分代码
    - 1、线程池配置代码（注意要加上 `@EnableAsync` 注解，使异步生效）
    - 2、`LogServiceImpl` 中 `save` 方法增加了 `@Async("logServiceExecutor")` 注解
- 通过如上两步，简单快捷实现方法异步处理，并且线程池与业务隔离

---

本文完。
