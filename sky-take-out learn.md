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

#### Bean

就是由spring管理的对象就是Bean

**Spring Bean的优点**

- **解耦（最关键）**：假如你的 `UserService` 依赖 `OrderService`，如果直接在代码里 `new OrderService()`，那这两个类就“硬绑定”在一起了。以后如果 `OrderService` 的构造函数变了，你所有 `new` 它的地方都要改。用了 IOC，你只依赖接口，具体用哪个实现类由 Spring 在外部配置决定，代码瞬间变得非常灵活。

  **解耦的本质是“隔离变化”。**

- **方便管理对象生命周期**：很多对象需要是单例的（比如数据库连接池），或者需要在使用前执行初始化、使用后执行销毁。如果自己 new，这些生命周期逻辑得写一堆重复代码。IOC 容器可以统一帮你管理这些“生老病死”。

## **框架就是作文模板**

**Spring 就是一个超级大的“对象工厂”+“工具包”，让你写代码时不用操心对象的创建和管理，只管专注写业务逻辑。**

Spring框架的核心功能有两个：

- Spring容器作为超级大工厂，负责创建、管理所有的Java对象，这些Java对象被称为Bean。
- Spring容器管理容器中Bean之间的依赖关系，Spring使用一种被称为"依赖注入"的方式来管理Bean之间的依赖关系。

## JVM类加载机制

#### 双亲委派机制

**双亲委派机制（Parent Delegation Model）** 是Java类加载器的一种工作模式：**当一个类加载器收到类加载请求时，它不会自己去尝试加载，而是把请求委派给它的父类加载器去完成，层层向上传递，直到顶层的启动类加载器。只有当父类加载器反馈无法加载时，子加载器才会尝试自己加载。**

**核心类加载器**

在JVM中，类加载器有明确的层级关系：

- **启动类加载器（Bootstrap ClassLoader）**：顶级，由C++实现，负责加载 `JAVA_HOME/lib` 下的核心类库（如 `rt.jar`）。
- **扩展类加载器（Extension ClassLoader）**：由Java实现，负责加载 `JAVA_HOME/lib/ext` 下的扩展类库。
- **应用程序类加载器（Application ClassLoader）**：由Java实现，负责加载用户类路径（Classpath）上的类，也就是我们自己写的代码。

**注意**：它们之间的父子关系不是继承，而是**组合**（通过 `parent` 属性关联）

```
+------------------------------------------------------+
|                   启动类加载器                         |
|           （Bootstrap ClassLoader）                   |
|    • C++ 实现，JVM 内置                              |
|    • 加载 JAVA_HOME/lib 目录下的核心类库             |
|    • 例如：rt.jar 中的 java.lang.* 等                |
+---------------------------+--------------------------+
                            |
                            |  parent（通过组合关系关联）
                            |
+---------------------------+--------------------------+
|                   扩展类加载器                         |
|           （Extension ClassLoader）                   |
|    • Java 实现，位于 sun.misc.Launcher$ExtClassLoader  |
|    • 加载 JAVA_HOME/lib/ext 目录下的扩展类库             |
+---------------------------+--------------------------+
                            |
                            |  parent（通过组合关系关联）
                            |
+---------------------------+--------------------------+
|                   应用程序类加载器                     |
|           （Application ClassLoader）                 |
|    • Java 实现，位于 sun.misc.Launcher$AppClassLoader |
|    • 加载 Classpath（类路径）下的类                   |
|    • 例如：我们自己写的业务代码 .class 文件           |
+---------------------------+--------------------------+
                            |
                            |  可以继续挂载自定义子加载器
                            |
+---------------------------+--------------------------+
|                   自定义类加载器                       |
|           （Custom ClassLoader）                      |
|    • 用户自定义，非必需                              |
|    • 可加载指定目录、网络、加密后的字节码等           |
+------------------------------------------------------+
```

JVM**类加载机制**

**加载 → 连接（验证→准备→解析）→ 初始化**

**堆里放对象，方法区放类**

#### 加载（Loading）

- **做什么**：找到类的二进制字节流（.class 文件、网络、JAR 包等），通过类全限定名获取，将其转换成方法区的数据结构，并在堆中生成一个 `java.lang.Class` 对象作为访问入口。
- **谁来做**：类加载器（就是上一轮说的双亲委派机制工作的阶段）。

#### 2. 连接（Linking）

##### 2.1 验证（Verification）

- **做什么**：确保字节流符合 JVM 规范，不会危害 JVM 安全。

##### 2.2 准备（Preparation）

- **做什么**：为**静态变量（类变量）** 分配内存，并设置**默认初始值**（不是代码里写的那个值）。

##### 2.3 解析（Resolution）

- **做什么**：将常量池中的**符号引用**替换为**直接引用**（内存中的实际地址）。
- **符号引用**：比如一个类引用了另一个类 `com.example.User`，这个只是字符串。
- **直接引用**：具体的内存偏移量或指针，能直接访问到目标类/方法/字段。
- **解析时机**：有的 JVM 会延迟到实际使用时才解析（即用即解析），不一定是连接阶段全部完成。

#### 3. 初始化（Initialization）

- **做什么**：真正执行类构造器 `<clinit>()` 方法，为静态变量赋**代码里写的值**，并执行静态代码块。
- **承接上例**：此时 `count` 才从 `0` 变为 `100`。

**总结：加载和验证把静态的。class文件转化为动态的类型数据，准备和初始化就是给数据分配内存，解析则是在当前类型和其他类型之间建立联系**

### 问

##### **准备阶段分配内存那加载阶段数据在哪里呀？**

加载阶段的数据还在JVM内存里，但处于未加工的原始字节流状态；进入准备阶段后，JVM会解析这个字节流，根据它的描述，在方法区（元空间）里给静态变量重新分配一块正式内存，并填上默认值。而加载阶段读进来的那些原始字节，在完成“翻译”任务后就被抛弃了，留下来的只有方法区里的那个正式结构。

| 维度         | 加载阶段                           | 准备阶段                   |
| :----------- | :--------------------------------- | :------------------------- |
| **数据位置** | JVM内存中（堆外或方法区的临时区）  | 方法区（元空间）的正式空间 |
| **数据形态** | `byte[]` 二进制流（Class文件原样） | JVM内部数据结构（C++对象） |

**类加载路径是“逻辑上的寻址根目录”，而文件系统的相对/绝对路径是“物理文件在磁盘上的真实位置”。**

- **类加载路径（Classpath）**：基准点是**类加载器（ClassLoader）的“类路径根目录”**。
  - 它不关心你的代码放在磁盘的哪个盘符，而是把 `target/classes/` 和所有依赖的 JAR 包逻辑上合并成一个“虚拟根目录”。
  - 只要在类路径根目录下，你写 `classpath:config/app.yml`，无论项目部署在 Windows 还是 Linux，无论启动目录在哪，都能找到。

**AOP面向切面编程**

把多个方法都要用到的功能比如说输出日志，抽象为一个切面，看作一个对象去处理就是面向切面编程

底层使用动态 代理机制去实现

会出现一个**代理对象**

在Spring AOP的语境下，当你对一个Bean应用了切面（比如加了`@Transactional`或`@Around`），Spring不会直接返回那个原始对象，而是返回一个**代理对象**给你。你调用的所有方法，实际上都是先调用了这个代理对象的方法，代理对象再在合适的时机去调用原始对象的方法。

实际编程中对代理对象进行编程









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

