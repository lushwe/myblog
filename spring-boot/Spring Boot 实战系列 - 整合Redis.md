# Spring Boot 实战系列 - 整合Redis

> Spring Boot 整合 Redis 超级简单，看下面代码就知道


## 首先，看 `pom.xml` 文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <!-- 这是我自己工程的父级 -->
    <parent>
        <artifactId>learn-spring-boot</artifactId>
        <groupId>com.lushwe.learn</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <modelVersion>4.0.0</modelVersion>
    <artifactId>spring-boot-data-redis</artifactId>

    <!-- 依赖jar如下 -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
    </dependencies>
</project>
```

## 其次，看 `application.properties` 文件
```properties
spring.redis.host=127.0.0.1
spring.redis.port=6379
spring.redis.password=xxx
```

## 最后，看启动类
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

/**
 * 说明：Redis测试启动类
 *
 * @author Jack Liu
 * @date 2020-08-14 14:29
 * @since 0.1
 */
@SpringBootApplication
public class RedisBootstrap {

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    public static void main(String[] args) {
        SpringApplication.run(RedisBootstrap.class, args);
    }

    @Component
    class RedisRunner implements ApplicationRunner {

        @Override
        public void run(ApplicationArguments args) throws Exception {
            String key = "learn:redis:lushwe";
            stringRedisTemplate.opsForValue().set(key, "Jack Liu");
            String value = stringRedisTemplate.opsForValue().get(key);
            System.out.println("value=" + value);
        }
    }
}
```

- 启动运行后，日志
```xml
value=Jack Liu
```

## 总结：
- `Spring Boot` 整合 `Redis` , 只引入了 `spring-boot-starter-data-redis` , 以及配置 `application.properties` 文件
- 得益于 `Spring Boot` 自动配置的功劳，项目中可以直接注入 `StringRedisTemplate` 进行使用，超级简单便捷有没有

---

#### 本文完
