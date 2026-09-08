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

# 最最笼统的后端书写任务

* 基础的配置和连接
* 定义测试接口并访问
* 登录接口
* 加JWT
* 加JWT过滤器



# 一 第一个网络接口和基础配置

## 后端项目的三层

- **Controller** —— **控制层**，负责接收请求和返回响应，写一些接口
- **Service** —— **业务逻辑层**，负责处理核心业务。
- **Mapper / DAO** —— **数据访问层**，负责操作数据库。

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

# 定义一个网络接口

下面三个都是通过注解来完成的

* 定义请求路径
* 定义请求类型
* 定义返回格式

其实是一个控制类，控制类是API接口（网络层面的接口不是Java语法的接口)

控制类是网络访问的接口

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

### 关于这三个注解

这三个注解是 Spring Boot 开发中最基础的三个注解，它们共同把一个普通的 Java 类变成了一个能接收 HTTP 请求的控制器

## 1. `@RestController` —— 标记这个类是“控制器”，且返回数据而不是页面

```
@RestController  // ← 这是类级别的注解
public class Test {
    // ...
}
```

**含义：** 告诉 Spring：“这个类是一个 **Web 控制器**，里面所有方法的返回值都要**直接写进 HTTP 响应体（Response Body）**，而不是跳转到一个页面。”

------

## 2. `@RequestMapping("/api")` —— 给这个类里的所有接口加一个“共同前缀”

```
@RequestMapping("/api")  // ← 类级别的路径
public class Test {
    // ...
}
```

**含义：** 告诉 Spring：“这个控制器里所有接口的 URL 前面，都要加上 `/api`。”

**好处：** 方便统一管理接口路径。比如以后想把所有接口从 `/api` 改成 `/v2/api`，只需要改这一行就行。

------

## 3. `@GetMapping("/test")` —— 标记这个方法处理 GET 请求，路径是 `/test`

```
@GetMapping("/test")  // ← 方法级别的注解
public String test() {
    return "hello";
}
```

**含义：** 告诉 Spring：“当客户端发送一个 **GET 请求**，访问路径为 **`/test`** 时，由这个方法处理。”

### 五种最常见的请求类型

我们可以用一个 **“图书馆管理系统”** 的例子来理解，假设我们要管理书籍（Book）数据：

| 请求类型   | 作用（你要干嘛）         | 比喻（图书馆）                               | 代码中的注解     |
| :--------- | :----------------------- | :------------------------------------------- | :--------------- |
| **GET**    | **查**询数据（只看不摸） | 查询书架上有哪些书，或查询某本书的详细信息。 | `@GetMapping`    |
| **POST**   | **新**增数据（创建）     | 购入一本新书，需要在系统里**新增**一条记录。 | `@PostMapping`   |
| **PUT**    | **修**改数据（全量覆盖） | 把一本旧书的信息**完全替换**成新的信息。     | `@PutMapping`    |
| **PATCH**  | **修**改数据（局部更新） | 只修改某本书的价格或位置，其他信息不动。     | `@PatchMapping`  |
| **DELETE** | **删**除数据             | 把一本破损的书从系统中**移除**。             | `@DeleteMapping` |

## reesult类

**用于统一返回值类型的类**

```java
package org.example.ai.spingboot.common;

import lombok.Data;

@Data
public class Result<T> {
    private String code;
    private String msg;
    private T data;
}
public static <T> Result<T> success() {
    Result<T> result = new Result<>();
    result.setCode("200");
    result.setMsg("success");
    return result;
}
```

- [x] ##### 成功写出一个接口并访问

## 访问地址的格式

```
http://       localhost       :8080         /api/test
  ① 协议       ② 服务器地址    ③ 端口       ④ 路径
```

### 路径：`/api/test`

- 这是你**在代码里写的路径**，由两部分拼成：

| 写在哪儿   | 代码                      | 路径            |
| :--------- | :------------------------ | :-------------- |
| 类上       | `@RequestMapping("/api")` | `/api`          |
| 方法上     | `@GetMapping("/test")`    | `/test`         |
| **拼起来** |                           | **`/api/test`** |

# Lombok

**Lombok 就是自动生成 getter/setter 等样板代码的工具，用一个注解代替一堆重复代码，让 Java 类更简洁**

## 在 Result 类中

```java
import lombok.Data;

@Data
public class Result<T> {
    private String code;
    private String msg;
    private T data;
}
```

这等价于你手动写了：

```java
public class Result<T> {
    private String code;
    private String msg;
    private T data;

    // 无参构造
    public Result() {}

    // getter
    public String getCode() { return code; }
    public String getMsg() { return msg; }
    public T getData() { return data; }

    // setter
    public void setCode(String code) { this.code = code; }
    public void setMsg(String msg) { this.msg = msg; }
    public void setData(T data) { this.data = data; }

    // toString
    @Override
    public String toString() {
        return "Result{code=" + code + ", msg=" + msg + ", data=" + data + "}";
    }

    // equals 和 hashCode（省略具体代码）
}
```

### Spring Boot 只会扫描**启动类所在的包及其子包**。

所以common和controller包都得在启动类下，不然扫描不到

# 二 登录接口

#### DTO数据传输对象

它的核心目的是在系统各层（如 Controller、Service、数据库）之间传递数据，通常是一个简单的 Java 对象，只包含属性及其对应的 getter/setter 方法，不包含任何业务逻辑。

## Spring validation

一套通过**注解**来优雅、高效地完成数据校验的规范及实现

@NotBlank

这个字段必须有实际内容，不能是空的，也不能是空格

### 登录接口

```java
@RestController
@RequestMapping("/api/user")
public class User {
    @PostMapping("/login")
    public Result<String> login(@Valid @RequestBody UserLoginCommandDTO commandDTO) {
        //校验+从请求体拿数据+存到UserLoginCommandDTO类型名字是commandDTO的变量中
        System.out.println(commandDTO.getUsername());
        System.out.println(commandDTO.getPassword());
        return null;
    }
}
```

# 三 校验异常处理

##### 参数校验异常处理代码

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 处理参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<String> handlerException(MethodArgumentNotValidException e) {
        // 处理异常数据的处理：获取所有校验失败的信息，用逗号拼接成字符串
        String message = e.getBindingResult()          // 获取校验结果对象
                          .getFieldErrors()            // 获取所有字段错误列表（List<FieldError>）
                          .stream()                    // 转换成流
                          .map(FieldError::getDefaultMessage) // 提取每条错误的 message
                          .collect(Collectors.joining(",")); // 用逗号拼接成字符串
        return Result.error(ResultCode.PARAM_ERROR.getCode(), message);
    }
}
```

##### e是 Spring 自动传入的异常对象

# 四 用户实体类

**数据库里的"用户表"映射成代码里的"用户对象"**。

## MyBatis-Plus 注解

```java
@Data
@TableName("user")
public class User {
    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    private String username;
    private String password;
    private String email;
    private String nickname;
    private String avatar;
    private String phone;
    private Integer gender;
    private LocalDate birthday;

    @TableField("user_type")
    private Integer userType;

    private Integer status;

    @TableField("created_at")
    private LocalDateTime createdAt;

    @TableField("updated_at")
    private LocalDateTime updatedAt;
}
```

## 类级别注解

### `@Data`

- **来源**：Lombok 库
- **作用**：自动生成 getter、setter、toString、equals、hashCode 等方法
- **效果**：你不用手动写 `getId()`、`setUsername()` 这些重复代码，Lombok 会在编译时帮你补全

------

### `@TableName("user")`

- **来源**：MyBatis-Plus
- **作用**：告诉 MyBatis-Plus，这个实体类对应数据库里的 **`user`** 表
- **为什么需要**：如果类名是 `User`，默认会去找 `user` 表，大小写不敏感时可以不写；但如果表名不同（比如 `t_user`），就必须用这个注解指定

------

## 字段级别注解

### `@TableId(value = "id", type = IdType.AUTO)`

- **来源**：MyBatis-Plus
- **作用**：标记这个字段是数据库表的**主键**
- **参数解释**：
  - `value = "id"`：对应数据库表里的 `id` 字段
  - `type = IdType.AUTO`：主键生成策略是**自增**（数据库自动生成，插入时不用手动赋值）
- **其他可选类型**：
  - `IdType.INPUT`：手动输入
  - `IdType.UUID`：自动生成 UUID
  - `IdType.ASSIGN_ID`：雪花算法生成（MyBatis-Plus 默认）

------

### `@TableField("user_type")`

- **来源**：MyBatis-Plus
- **作用**：标记字段映射到数据库的哪一列
- **为什么这里需要**：因为 Java 字段名是 `userType`（驼峰命名），但数据库列名是 `user_type`（下划线命名），所以需要用这个注解指定映射关系
- **如果不写会怎样**：MyBatis-Plus 默认开启驼峰转下划线，所以其实 `userType` → `user_type` 能自动转换，这里写出来是为了明确指定

------

### `@TableField("created_at")` 和 `@TableField("updated_at")`

- 同理，映射数据库的 `created_at` 和 `updated_at` 列
- 这两个字段通常配合数据库的 `CURRENT_TIMESTAMP` 和 `ON UPDATE CURRENT_TIMESTAMP` 使用，或者由代码自动填充

------

## 其他没有注解的字段

像 `username`、`password`、`email` 这些字段**没有注解**，但 MyBatis-Plus 会通过**驼峰转下划线**自动映射：

- `username` → `username`（一样）
- `nickname` → `nickname`（一样）
- `createdAt` 如果没写 `@TableField`，也会自动转成 `created_at`

所以这些没注解的字段也能正常工作。

## `@Pattern` 注解

`@Pattern` 是 **Java Bean Validation（JSR-303/JSR-380）** 规范中的一个注解，用于**验证字符串字段是否符合指定的正则表达式规则**。

## ==依赖注入==

```
@Resource
    private JwtUtil jwtUtil;
```

`@Resource` 是 **JSR-250 规范**中定义的注解，用于**声明一个依赖项**，告诉 Spring 容器：**“这个字段需要一个 Bean，请在容器中帮我找到并赋值”**。

`JwtUtil` 是一个 **Token 生成与解析工具类**。

| 组成部分    | 技术含义                                                     |
| :---------- | :----------------------------------------------------------- |
| `@Resource` | 这是一个**注入点标记**。Spring 在启动时扫描到这个注解，就会执行依赖查找和注入逻辑。 |
| `private`   | 访问修饰符。虽然字段是私有的，但 Spring 通过**反射（Reflection）** 机制绕过访问限制，强行赋值。 |
| `JwtUtil`   | **依赖类型**。声明这个字段需要的是一个类型为 `JwtUtil` 的 Bean。 |
| `jwtUtil`   | **变量名**。在 `@Resource` 的默认匹配策略中，这个名称会被用作 **Bean 的名称** 进行查找。 |

### ⚙️ 执行流程（Spring 启动时）

1. **扫描**：Spring 在启动时扫描所有 Bean，发现 `AuthController` 类中有一个标记了 `@Resource` 的字段。
2. **查找**：Spring 根据 `@Resource` 的匹配规则，在容器中查找符合条件的 Bean。
3. **赋值**：找到目标 Bean 后，通过反射将该 Bean 的引用赋值给 `jwtUtil` 字段。
4. **注入完成**：`AuthController` 对象内部的 `jwtUtil` 字段现在持有了一个有效的 `JwtUtil` 实例，可以在业务方法中调用。

# 自定义业务异常处理

# Spring-Security

## 配置类和普通类的区别

|                | 普通类（`@Service`/`@Controller`） | 配置类（`@Configuration`）                          |
| :------------- | :--------------------------------- | :-------------------------------------------------- |
| **作用**       | 实现业务逻辑                       | 定义 Bean 的创建规则                                |
| **谁创建它**   | Spring 自动扫描并创建              | Spring 自动扫描并创建                               |
| **里面有什么** | 业务方法（增删改查）               | `@Bean` 方法（返回对象）                            |
| **典型例子**   | `UserService`、`UserController`    | `SecurityConfig`、`RedisConfig`、`DataSourceConfig` |

# JWT(**JSON Web Token**) 认证 

**无状态的身份认证**

一个 JWT 看起来像这样：

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE1MTYyMzkwMjJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

它是三个部分用 `.` 拼接的：

```
xxxxx.yyyyy.zzzzz
```

| 部分     | 名称              | 内容                          | 能看懂吗？             |
| :------- | :---------------- | :---------------------------- | :--------------------- |
| 第一部分 | Header（头部）    | 加密算法信息                  | Base64编码，解码后可读 |
| 第二部分 | Payload（载荷）   | 用户信息（如 userId、用户名） | Base64编码，解码后可读 |
| 第三部分 | Signature（签名） | 前两部分 + 密钥 加密生成      | 不可伪造               |

**关键点：JWT 的内容是可见的（Base64编码），只是不能被篡改。** 所以不要在 JWT 里存密码等敏感信息！

## JWT(**JSON Web Token**) 认证涉及的代码

| 你问的"代码要在哪里写" | 答案                                  |
| :--------------------- | :------------------------------------ |
| JWT 生成/解析逻辑      | `service/JwtService.java`             |
| JWT 拦截过滤器         | `config/JwtAuthenticationFilter.java` |
| 登录接口               | `controller/UserController.java`      |
| 登录请求对象           | `DTO/command/LoginRequest.java`       |
| 登录验证逻辑           | `service/UserService.java`            |
| 注册过滤器             | `config/SecurityConfig.java`（修改）  |
| JWT 配置               | `application.yml`                     |

**注意：** 如果 `UserMapper` 里还没有 `findByUsername` 方法，记得在 `UserMapper.java` 里加上

- [ ] 先在xml文件里写好JWT需要约定的数据

- [ ] 再在config写好配置类

- [ ] 创建utill获取bean，写token生成工具

- [ ] 构建响应DTO，用UserConvert 

- [ ] 在service写调用逻辑

  UserConvert是一个转换器，将数据库里的数据转换成返回给前端的人能看懂的信息

  为什么之前login DTO不用转换器？

  **只有"需要从 Entity 转换"的 DTO 才需要转换器。**
  **如果 DTO 只是"接收前端数据"或"纯 Token 响应"，就不需要。**

  login DTO它只是"接收容器"，前端传什么就接什么，不需要从任何其他对象转换过来。

  但是从数据库里拿出来的数据格式和前端要的不匹配，就需要转换器

- [ ] 111
