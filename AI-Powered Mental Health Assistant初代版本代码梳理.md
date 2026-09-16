# AI-Powered Mental Health Assistant初代版本代码梳理

## 概念清单

#### ==Mapper==：负责对接数据库的数据存取

#### ==wrapper==：`Wrapper` 是 MyBatis-Plus 里的一个**核心概念**，中文叫 **“条件构造器”** 或 **“包装器”**。

#### 你可以把它想象成一张 **“查询条件清单”**：你把想要的条件一条一条写上去，MyBatis-Plus 帮你把它翻译成 SQL 的 `WHERE`、`ORDER BY` 等部分。

#### ==service层==实现具体方法

#### ==controller层==调用service层写的方法

#### ==Entity==实体类，数据库表的镜像

#### ==DTO==层与层之间的数据传输对象

![image-20260916204721030](C:\Users\ThinkPad\AppData\Roaming\Typora\typora-user-images\image-20260916204721030.png)

#### ==command==从前端到后端

#### ==response==从后端返给前端

**==`config` 包==是整个项目的“控制中心”。它负责安全认证（登录、权限、JWT）、AI 模型对接（硅基流动 API 配置）、以及其他全局设置。它不处理具体业务，只为业务层提供“可用的工具和规则”。**