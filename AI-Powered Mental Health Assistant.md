# AI-Powered Mental Health Assistant

## 项目技术栈

## AI/java/spring boot/springAI/vue3

| 技术                                       | 版本           | 说明                 |
| :----------------------------------------- | :------------- | :------------------- |
| Java                                       | 17             | 编程语言             |
| Spring Boot                                | 3.4.1          | 后端框架             |
| Spring AI (spring-ai-starter-model-openai) | 1.0.0-SNAPSHOT | AI生成框架（OpenAI） |
| MyBatis Plus                               | 3.5.7          | ORM框架              |
| spring-boot-starter-security               | -              | 安全框架             |
| Spring Data Redis                          | -              | Redis缓存            |
| MySQL                                      | -              | 关系型数据库         |
| Spring Validation                          | -              | 参数校验             |
| Java JWT                                   | 4.4.0          | JWT认证              |
| Spring AOP                                 | -              | 切面编程             |
| Spring Mail                                | -              | 邮件发送             |
| hutto-all                                  | -              | Java工具类库         |

**Node.js**

**它不是一门新语言，而是 JavaScript 的运行环境**

## 新建项目需要的依赖项

- **Lombok**

  一个编译时注解处理器，通过注解自动生成 Java 类的样板代码

- **Spring Web**

  - Spring Boot 中用于构建 Web 应用（包括 RESTful API）的起步依赖。
  - **核心内容**：内嵌了 **Tomcat** 服务器，并集成了 **Spring MVC** 框架。

- **MySQL Driver**

  - Java 连接 MySQL 数据库的驱动程序（JDBC Driver），具体实现是 `mysql-connector-j`。
  - **解决什么问题**：它是 Java 应用和 MySQL 数据库之间的"翻译官"，让 Java 代码能够通过 JDBC 协议发送 SQL 语句并读取查询结果。

- **Spring Data JDBC**

  - Spring 家族中基于 JDBC 的轻量级数据访问框架，属于 Spring Data 项目的一部分。
  - **解决什么问题**：让你通过注解（如 `@Table`、`@Id`、`@Column`）将 Java 实体类直接映射到数据库表，省去手写大量 JDBC 模板代码（如 `PreparedStatement`、`ResultSet` 解析）。

### Spring Boot 配置文件格式

Spring Boot 支持两种配置文件格式：

| 格式           | 文件名                   | 特点                                         |
| :------------- | :----------------------- | :------------------------------------------- |
| **properties** | `application.properties` | 键值对平铺，适合简单配置，但长配置会重复冗余 |
| **YAML**       | `application.yml`        | 树形层级结构，可读性更强，适合复杂配置       |

## `application.yml`----***Spring Boot 的核心配置文件***

## application.yml的内容

```
spring:
  datasource:      # 数据源配置（数据库连接）
  jdbc:            # JDBC 配置
  redis:           # Redis 配置（如果你用了）
  rabbitmq:        # 消息队列配置（如果你用了）
  mail:            # 邮件配置（如果你用了）
```

spring是大文件夹，里面放置各种子配置

## 配置数据库连接

```java
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/你的数据库名?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: 你的MySQL密码
    driver-class-name: com.mysql.cj.jdbc.Driver
```

# 定义一个接口

下面三个都是通过注解来完成的

* 定义请求路径
* 定义请求类型
* 定义返回格式

````java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController//返回格式
@RequestMapping("/api")//访问路径
public class Test {
    @GetMapping("/test")//请求类型
    public String test() {
        return "hello world";
    }
}
````



