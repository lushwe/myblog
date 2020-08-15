# Spring Boot 实战系列 - 发送邮件

> 使用 `spring boot` 快速开发发送邮件功能

## 准备工作
- `pom.xml` 文件依赖如下
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
    </dependency>
    <!-- 邮件依赖 start -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>
    <!-- 邮件依赖 end -->
</dependencies>
```

- `application.properties` 配置文件如下
```properties
spring.application.name=spring-boot-mail

# Spring Boot 整合Mail配置
spring.mail.host=smtp.qq.com
spring.mail.username=xxx@qq.com
# qq邮箱授权码
spring.mail.password=xxxxxx
spring.mail.default-encoding=UTF-8

# 邮件发送相关配置
mail.address.to=xxx@qq.com
mail.address.from=xxx@qq.com
```

- `MailService` 代码如下
```java
@Slf4j
@Service
public class MailService {

    @Autowired
    private JavaMailSender javaMailSender;

    @Value("${mail.address.from}")
    private String from;

    public void sendSimpleMail(String[] to, String subject, String text) {
        SimpleMailMessage simpleMailMessage = new SimpleMailMessage();
        simpleMailMessage.setSubject(subject);
        simpleMailMessage.setText(text);
        simpleMailMessage.setSentDate(new Date());
        simpleMailMessage.setTo(to);
        simpleMailMessage.setFrom(from);
        log.info("发送普通邮件，参数：{}", JSON.toJSONString(simpleMailMessage));
        javaMailSender.send(simpleMailMessage);
    }
}
```

- `Application` 启动类代码如下
```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {

        // 启动
        SpringApplication.run(Application.class, args);
    }

}
```

- `MailServiceTest` 测试类代码如下
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class MailServiceTest {

    @Autowired
    private MailService mailService;

    @Value("${mail.address.to}")
    private String to;

    @Test
    public void testSendSimpleMail() {

        String[] tos = to.split(",");
        mailService.sendSimpleMail(tos, "主题-测试", "内容-测试");
    }
}
```

## 说明
- 配置中 `spring.mail.password` 为授权码，本示例使用QQ邮箱，需要自行去申请授权码
- 只要依赖 `spring-boot-starter-mail` ，可以直接使用 `JavaMailSender` 实例（因为 `Spring Boot` 自动配置）

---

本文完。
