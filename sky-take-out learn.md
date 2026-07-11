# sky-take-out learn

**整个项目结构**

```
sky-take-out/
├── .idea/                          # IDEA 项目配置文件
├── .vscode/                        # VSCode 配置文件
├── demo/                           # 示例模块（可能是测试用）
├── sky-common/                     # 公共模块（通用工具、常量、异常等）
│   ├── src/
│   │   └── main/
│   │       └── java/
│   │           └── com/
│   │               └── sky/
│   │                   ├── constant/          # 常量类
│   │                   ├── context/           # 上下文对象（如线程上下文）
│   │                   ├── enumeration/       # 枚举类
│   │                   ├── exception/         # 自定义异常
│   │                   ├── json/              # JSON 序列化相关
│   │                   ├── properties/        # 配置文件属性类（@ConfigurationProperties）
│   │                   ├── result/            # 统一返回结果封装
│   │                   └── utils/             # 工具类
│   └── pom.xml
│
├── sky-pojo/                       # 实体类模块（POJO）
│   ├── src/
│   │   └── main/
│   │       └── java/
│   │           └── com/
│   │               └── sky/
│   │                   ├── dto/              # Data Transfer Object（数据传输对象）
│   │                   ├── entity/           # 数据库实体类（对应表结构）
│   │                   └── vo/               # View Object（视图对象，返回前端）
│   └── pom.xml
│
├── sky-server/                     # 主服务模块（启动类、控制器、业务逻辑）
│   ├── src/
│   │   └── main/
│   │       ├── java/
│   │       │   └── com/
│   │       │       └── sky/
│   │       │           ├── annotation/       # 自定义注解
│   │       │           ├── aspect/           # AOP 切面类
│   │       │           ├── config/           # 配置类（@Configuration）
│   │       │           ├── controller/       # 控制器层
│   │       │           │   ├── admin/        # 管理端接口
│   │       │           │   ├── notify/       # 通知相关接口
│   │       │           │   └── user/         # 用户端接口
│   │       │           ├── handler/          # 处理器（如异常处理器、WebSocket处理器）
│   │       │           ├── interceptor/      # 拦截器
│   │       │           ├── mapper/           # MyBatis Mapper 接口（DAO层）
│   │       │           ├── service/          # 业务逻辑层（Service）
│   │       │           │   ├── impl/         # Service 实现类（通常在此子包）
│   │       │           ├── task/             # 定时任务（如 Spring Scheduled）
│   │       │           ├── websocket/        # WebSocket 相关
│   │       │           └── SkyApplication    # Spring Boot 启动类
│   │       └── resources/                    # 资源文件（yml、xml、静态资源等）
│   └── pom.xml
│
├── .gitignore                      # Git 忽略文件
├── LICENSE                         # 开源许可证
├── pom.xml                         # 父 POM（Maven 多模块管理）
└── README.md                       # 项目说明文档
```



#### **constant**

**常量类：专门用来集中存放常量（不会改变的值）的类**

**全局固定值，便于整体修改**

#### **context**

上下文对象是**在程序运行过程中，携带当前环境、状态、用户信息、请求参数等数据的载体**。它像一个“工具箱”，在调用链中传递，让不同层级的代码都能获取到所需的信息。

上下文对象”就是**把当前执行环境的所有必要信息打包成一个对象**

#### enumeration

枚举类，就是枚举

#### exception

异常，特殊情况==看看异常里面写的啥==

#### json

`json/` 目录里的类，封装的**不是JSON数据本身**，而是**操作JSON数据的工具**，就像：

| 比喻                        | 说明                                       |
| :-------------------------- | :----------------------------------------- |
| **JSON**                    | 像一张"表格"（数据格式）                   |
| **json/ 目录**              | 像"Excel工具箱"（处理表格的工具集合）      |
| **JacksonConfig**           | 设置Excel的"默认字体/日期格式"（全局配置） |
| **JsonUtils**               | "复制/粘贴/转换"的快捷键（封装常用操作）   |
| **LocalDateTimeSerializer** | 告诉Excel"日期怎么显示"（自定义规则）      |

- 它不存JSON数据
- 它提供了一套**统一的工具和方法**，让整个项目能够**规范地、一致地**处理JSON的序列化和反序列化

#### **properties配置文件属性类**

**配置文件是一个文件，存放配置信息**

```
# 这是配置文件本身（放在 src/main/resources/ 下）
app:
  name: 我的项目
  version: 1.0.0
  port: 8080
```

**1. `properties` 包放的是“配置映射类”，不是配置文件本身。**

**2. 配置文件（.yml）是“数据源”，映射类（XxxProperties）是“数据容器”。**

**3. Spring 负责把配置文件里的数据“倒进”映射类里，业务代码只管用这个类就行。**

**三者关系**

```
┌─────────────────────────────────────────────────────────┐
│  配置文件（原料）                                        │
│  src/main/resources/application.yml                     │
│                                                         │
│  app:                                                   │
│    name: 我的项目    ←───────┐                           │
│    version: 1.0.0   ←───────┤  Spring 自动读取          │
│    port: 8080       ←───────┤  并赋值给                 │
└─────────────────────────────────│───────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────┐
│  配置映射类（模具/容器）                                 │
│  src/main/java/.../properties/AppProperties.java       │
│                                                         │
│  @ConfigurationProperties(prefix = "app")              │
│  public class AppProperties {                           │
│      private String name;      ← 装进 app.name         │
│      private String version;   ← 装进 app.version      │
│      private int port;         ← 装进 app.port         │
│  }                                                      │
└─────────────────────────────────────────────────────────┘
                                  │
                                  │  @Autowired 注入
                                  ▼
┌─────────────────────────────────────────────────────────┐
│  业务代码（使用者）                                      │
│  src/main/java/.../service/DemoService.java             │
│                                                         │
│  @Autowired                                              │
│  private AppProperties appProperties;                    │
│                                                         │
│  appProperties.getName()  → "我的项目"                   │
│  appProperties.getPort()   → 8080                       │
└─────────────────────────────────────────────────────────┘
```

####  result/            # 统一返回结果封装

#### utils/             # 工具类

不依赖任何业务，纯功能函数，可全项目复用









**UserController**

```java
package com.sky.controller.user;

import java.util.HashMap;
import java.util.Map;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.sky.constant.JwtClaimsConstant;
import com.sky.dto.UserLoginDTO;
import com.sky.entity.User;
import com.sky.properties.JwtProperties;
import com.sky.result.Result;
import com.sky.service.UserService;
import com.sky.utils.JwtUtil;
import com.sky.vo.UserLoginVO;

import lombok.extern.slf4j.Slf4j;

@RestController
@RequestMapping("/user/user")
@Slf4j
public class UserController {
    @Autowired
    private UserService userService;

    @Autowired
    private JwtProperties jwtProperties;

    /**
     * 微信登录
     *
     * @param userLoginDTO
     * @return
     */
    @PostMapping("/login")
    public Result<UserLoginVO> login(@RequestBody UserLoginDTO userLoginDTO) {
        log.info("用户登录： userLoginDTO:{}", userLoginDTO);
        User user = userService.wxLogin(userLoginDTO);

        // 生成jwt令牌
        Map<String, Object> claims = new HashMap<>();
        claims.put(JwtClaimsConstant.USER_ID, user.getId());
        String token = JwtUtil.createJWT(jwtProperties.getUserSecretKey(), jwtProperties.getUserTtl(), claims);
        UserLoginVO userLoginVO = UserLoginVO.builder()
                .id(user.getId())
                .openid(user.getOpenid())
                .token(token)
                .build();

        return Result.success(userLoginVO);
    }
}

```

#### 概念

**注解**：给程序贴标签，它不会直接改变程序的运行逻辑，但工具或框架读到了这个标签，就会在编译时或运行时帮你做特定的事情。

**一个标准的Java注解由三部分组成：**

- **注解本身**：用 `@interface` 定义，比如 `public @interface Controller {}`
- **元注解**：贴在注解上的注解（用来限制这个注解能贴在哪里、活多久）。
  - `@Target`：规定这个注解能贴在**哪里**（是贴在类上、方法上，还是字段上？）
  - `@Retention`：规定这个注解**活多久**（是只在源代码里存在，还是在编译后的class文件里，还是在程序运行时也能通过反射读取？）

**元数据**

> **元数据就是“描述数据的数据”。**

它不是数据本身，而是告诉你**这份数据是什么、从哪里来、有什么特征、该怎么处理**的信息。

**一个极简的例子：**

- **数据**：一张照片（`IMG_2026.jpg`）
- **元数据**：这张照片的拍摄时间、地理位置、相机型号、文件大小——这些信息不改变照片本身，但帮你理解和管理这张照片。

**注解是产生元数据的一种方式，但元数据不只有注解这一种来源。**

**`@RestController` 标记一个类为 REST 风格的控制器，它里面的所有方法返回的数据（如 JSON/XML）会直接写入 HTTP 响应体，而不是跳转到页面。**

