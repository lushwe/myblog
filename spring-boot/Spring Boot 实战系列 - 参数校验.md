# Spring Boot 实战系列 - 参数校验

> Spring Boot 官网关于数据校验，只有寥寥几句，而且例子也相当简单，如下：

```java
@Service
@Validated
public class MyBean {

  public Archive findByCodeAndAuthor(@Size(min = 8, max = 10) String code,
      Author author) {
    ...
  }

}
```
但在使用过程中，还是遇到一些问题，下面记录下使用记录，以防以后忘记。

#### 首先看下 `pom.xml` 文件，详细如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>learn-spring-boot</artifactId>
        <groupId>com.lushwe.learn</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>spring-boot-exception</artifactId>

    <name>spring-boot-exception</name>
    <!-- FIXME change it to the project's website -->
    <url>http://www.example.com</url>

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
</project>
```

#### 实例一
```java
@Validated
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public String getUserById(@Min(value = 1, message = "参数不合法") @PathVariable("id") Long id) {
        return "zhangsan";
    }
}
```

- 注：
    - `UserController` 类上面需要加 `@Validated` 注解

#### 实例二
```java
@Validated
@RestController
public class UserController {

    @GetMapping(value = "/users")
    public String listUsers(@Valid UserReq userReq, Errors errors) {
        return "";
    }
}

@Data
public class UserReq {

    @NotNull(message = "不能为空")
    private Long userId;

}
```

- 注：
    - `UserReq` 类上面需要加 `@Valid` 注解，否则 `UserReq` 类里面的校验不生效
    - `UserReq` 类后面需要紧跟 `Errors` 类，用于接收错误，否则会一直报 `400` 错误

#### 本文完

