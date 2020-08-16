# Spring Boot 实战系列 - 任务调度

> 日常工作中经常需要在指定的时间做一些事情，比如下午18点发一份邮件等等，可以使用 `spring boot` 快速开发任务调度类用于处理类似需求

## 准备工作
- `pom.xml` 依赖如下
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

- `ScheduledTask` 代码如下
```java
@Component
public class ScheduledTask {

    @Scheduled(initialDelay = 100, fixedRate = 300)
    public void scheduled() {
        System.out.println("Task任务执行时间：" + LocalDateTime.now());
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Scheduled(fixedDelay = 3000)
    public void testDelay() {
        System.out.println("Dealy任务执行时间：" + LocalDateTime.now());
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Scheduled(cron = "0/1 * * * * ?")
    public void testCorn() {
        System.out.println("Corn任务执行时间：" + LocalDateTime.now());
    }
}
```

- `Bootstrap` 相关代码如下
```java
@EnableScheduling
@SpringBootApplication
public class Bootstrap {

    public static void main(String[] args) {
        SpringApplication.run(Bootstrap.class, args);
    }

}
```

## 总结
- 从代码可以看出，一共增加了两部分代码
    - 1、任务调度类：`ScheduledTask` ，需要加 `@Component` 注解
    - 2、启动类 `Bootstrap` 需要增加 `@EnableScheduling` 注解用于开启任务调度
- 该方案仅适合一些无关紧要的定时任务，对于要求比较高的任务最好使用分布式任务调度框架，如 `elastic-job`

---

本文完。
