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

