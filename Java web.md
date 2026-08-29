# Java web

#### 就是做网页

**JavaWeb 应用程序的三大核心组成部分及其职责：**

1. **网页（前端/展现层）**：展现数据
2. **JavaWeb程序（后端/逻辑处理层）**：作为整个应用的核心大脑，负责接收网页发来的请求，进行具体的业务逻辑运算和处理（如用户登录验证、数据计算等），并将处理结果返回给网页。
3. **数据库（数据存储层）**：负责持久化地存储和管理应用中的所有数据，为 JavaWeb 程序提供数据的增删改查（CRUD）服务。

# MySQL

#### 常用SQL语句

### 1. 查询（DQL）—— 重中之重

```sql
-- 基础查询
SELECT * FROM 表名;
SELECT 列1, 列2 FROM 表名;
-- 去除重复记录
SELECT DISTINCT 字段名 FROM 用户表


-- 条件查询
SELECT * FROM 用户表 WHERE 年龄 > 18;

-- 模糊查询
SELECT * FROM 用户表 WHERE 姓名 LIKE '%张%';

-- 排序
SELECT * FROM 用户表 ORDER BY 年龄 DESC;   -- 降序
-- ASC 升序（默认排序）
-- 分组查询
SELECT 性别, COUNT(*) FROM 用户表 GROUP BY 性别;

-- 分页查询
SELECT * FROM 表名 LIMIT 起始索引, 每页条数;

-- 多表联查（最常用）
SELECT u.姓名, o.订单金额 
FROM 用户表 u 
JOIN 订单表 o ON u.id = o.用户id;
```

### 2. 新增（DML）

```sql
INSERT INTO 用户表 (姓名, 年龄) VALUES ('张三', 25);
```

### 3. 修改（DML）

```sql
UPDATE 用户表 SET 年龄 = 26 WHERE 姓名 = '张三';
```

### 4. 删除（DML）

```sql
DELETE FROM 用户表 WHERE 姓名 = '张三';
```

### 5. 建表（DDL）

```sql
CREATE TABLE 用户表 (
    id INT PRIMARY KEY AUTO_INCREMENT,   -- 主键自增
    姓名 VARCHAR(50),
    年龄 INT,
    创建时间 DATETIME
);
```

==注意==MySQL是先写字段名再写数据类型，和Java是反着来的

![image-20260825214518120](C:\Users\ThinkPad\AppData\Roaming\Typora\typora-user-images\image-20260825214518120.png)

DOUBLE(总长度，小数点后保留的位数)

## 聚合函数

| 函数            | 作用                       | 返回值类型   |
| :-------------- | :------------------------- | :----------- |
| **COUNT(\*)**   | 统计总行数（所有记录数）   | 数字（整数） |
| **COUNT(列名)** | 统计该列 **非空** 值的数量 | 数字（整数） |
| **SUM(列名)**   | 计算该列数值的总和         | 数字         |
| **AVG(列名)**   | 计算该列数值的平均值       | 数字         |
| **MAX(列名)**   | 找出该列的最大值           | 与列类型一致 |
| **MIN(列名)**   | 找出该列的最小值           | 与列类型一致 |

##  **约束的概念**

- 约束是作用于表中列上的规则，用于限制加入表的数据
- 约束的存在保证了数据库中数据的正确性、有效性和完整性

## 约束类型

| 约束类型       | 关键字        | 作用                                             | 示例                                                     |
| :------------- | :------------ | :----------------------------------------------- | :------------------------------------------------------- |
| **主键约束**   | `PRIMARY KEY` | 唯一标识表中的每一行，**非空且唯一**             | `id INT PRIMARY KEY`                                     |
| **唯一约束**   | `UNIQUE`      | 该列的值 **不能重复**，但可以为 NULL             | `手机号 VARCHAR(11) UNIQUE`                              |
| **非空约束**   | `NOT NULL`    | 该列 **不能为空**（必须有值）                    | `姓名 VARCHAR(20) NOT NULL`                              |
| **默认值约束** | `DEFAULT`     | 如果插入时没指定值，则自动填入默认值             | `状态 INT DEFAULT 0`                                     |
| **外键约束**   | `FOREIGN KEY` | 表与表之间的关联，保证 **引用完整性**            | `用户id INT, FOREIGN KEY (用户id) REFERENCES 用户表(id)` |
| **检查约束**   | `CHECK`       | 限制列的值必须满足某个条件（MySQL 支持但不强制） | `年龄 INT CHECK (年龄 >= 18)`                            |

外键约束例子

```mysql
-- 创建学生表（父表）
CREATE TABLE students (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    class VARCHAR(20)
);

-- 创建成绩表（子表），添加外键
CREATE TABLE scores (
    id INT PRIMARY KEY AUTO_INCREMENT,
    student_id INT,
    subject VARCHAR(30),
    score INT,
    -- 外键约束：student_id 引用 students 表的 id
    CONSTRAINT fk_student FOREIGN KEY (student_id) REFERENCES students(id)
    -- CONSTRAINT 名称 FOREIGN KEY (本表字段) REFERENCES 父表(字段) 
    -- [ON DELETE 动作] [ON UPDATE 动作]
    ON DELETE CASCADE   -- 父表删除时，子表级联删除
    ON UPDATE CASCADE   -- 父表更新时，子表级联更新
);
```

| **不写** ON DELETE / ON UPDATE | **阻止删除/更新**（默认是 RESTRICT） |
| ------------------------------ | ------------------------------------ |
| **写了** ON DELETE CASCADE     | 允许级联删除                         |
| **写了** ON UPDATE CASCADE     | 允许级联更新                         |

![image-20260826115641107](C:\Users\ThinkPad\AppData\Roaming\Typora\typora-user-images\image-20260826115641107.png)

##### 数据库设计

* 有哪些表
* 表里有哪些字段
* 表和表之间有什么关系

### 表关系

一对多：从表建立外键关联主表

多对多：建立第三张中间表，中间表至少包含两个外键，分别关联两方主键

一对一：常见于表拆分，将一个实体中经常使用的字段放一张表，不经常使用的字段放另一张表，用于提升查询性能，在任意一方加入外键，另一方关联主键，并且设置外键为唯一（UNIQUE)

# 多表查询

* ## 连接查询

  * 内连接：相当于查询A B交集数据

    ### 内连接查询语法

    ```mysql
    -- 隐式内连接
    SELECT 字段列表 FROM 表1, 表2... WHERE 条件;
    
    -- 显示内连接
    SELECT 字段列表 FROM 表1 [INNER] JOIN 表2 ON 条件;
    ```

  * 外连接

    * 左外连接：查询A表和交集部分数据
    * 右外连接

    #### 外连接查询语法

    ```mysql
    -- 左外连接
    SELECT 字段列表 FROM 表1 LEFT [OUTER] JOIN 表2 ON 条件;
    
    -- 右外连接
    SELECT 字段列表 FROM 表1 RIGHT [OUTER] JOIN 表2 ON 条件;
    ```

* ## 子查询

  嵌套查询

  1. 子查询根据查询结果不同，作用不同：

     - 单行单列：作为条件值，使用 = != < 等进行比较判断

     ```mysql
     SELECT 字段列表 FROM 表 WHERE 字段名 = (子查询);
     ```

     

     - 多行单列：作为条件值，使用 in 等关键字进行条件判断

     ```mysql
     SELECT 字段列表 FROM 表 WHERE 字段名 in (子查询);
     ```

     

     - 多行多列：作为虚拟表

     ```mysql
     SELECT 字段列表 FROM (子查询) WHERE 条件;
     ```

  ##### 例子

  ```mysql
  -- 查询工资高于猪八戒的员工信息
  select * from emp;
  
  -- 1. 查询猪八戒的工资
  select salary from emp where name = '猪八戒';
  
  -- 2. 查询工资高于猪八戒的员工信息
  select * from emp where salary > 3600;
  
  select * from emp where salary > (select salary from emp where name = '猪八戒');
  ```

  

# 事务

**事务是一组SQL操作的逻辑单元，要么全部成功，要么全部失败**

```mysql
-- 开始事务
START TRANSACTION;
-- 或
BEGIN;

-- 执行SQL操作
UPDATE account SET money = money - 100 WHERE name = '张三';
UPDATE account SET money = money + 100 WHERE name = '李四';

-- 提交事务（确认操作）
COMMIT;

-- 或回滚事务（撤销操作）
ROLLBACK;
```

## 四大特征总览(ACID)

| 特征       | 英文        | 含义                                             | 一句话记忆             |
| :--------- | :---------- | :----------------------------------------------- | :--------------------- |
| **原子性** | Atomicity   | 事务是一个不可分割的整体，要么全成功，要么全失败 | 一荣俱荣，一损俱损     |
| **一致性** | Consistency | 事务前后，数据状态保持逻辑一致                   | 数据不能乱，规则不能破 |
| **隔离性** | Isolation   | 多个事务互不干扰，并发执行如同串行               | 你干你的，我干我的     |
| **持久性** | Durability  | 事务提交后，数据永久保存，即使宕机也不丢失       | 一旦提交，永不磨灭     |

# JDBC（Java数据库连接）

## 用Java语言操作关系型数据库的一套API

通过JDBC让同一套Java代码操作不同的关系型数据库

接口就是规则

![image-20260826123919501](C:\Users\ThinkPad\AppData\Roaming\Typora\typora-user-images\image-20260826123919501.png)

## JDBC 快速入门

### 步骤

1. 创建工程，导入驱动jar包

   - mysql-connector-java-5.1.48.jar

2. 注册驱动

   ```mysql
   Class.forName("com.mysql.jdbc.Driver");
   ```

3. 获取连接

   ```mysql
   Connection conn = DriverManager.getConnection(url, username, password);
   ```

4. 定义SQL语句

   ```mysql
   String sql = "update...";
   ```

5. 获取执行SQL对象

   ```mysql
   Statement stmt = conn.createStatement();
   ```

6. 执行SQL

   ```mysql
   stmt.executeUpdate(sql);
   ```

7. 处理返回结果

8. 释放资源

# API详解

- **DriverManager**：**驱动管理类**（或驱动管理器）
- **Connection**：**数据库连接对象**（或连接接口）
- **Statement**：**执行SQL语句对象**（或语句执行器）
- **ResultSet**：**结果集对象**（或查询结果集）
- **PreparedStatement**：**预编译SQL执行对象**（或预编译语句对象）

# DriverManager

* 注册驱动
* 管理数据库连接

```mysql
// 建立连接的核心方法
DriverManager.getConnection(String url, String user, String password);
```

URL写法

jdbc:mysql://ip地址(域名):端口号/数据库名称?参数键值对1&参数键值对2...

# Connection

* 获取执行SQL的对象
* 管理事务

#### 获取执行 SQL 的对象

- **普通执行SQL对象**

Statement createStatement()

* ##### 预编译SQL的执行SQL对象：防止SQL注入

PreparedStatement prepareStatement(sql)

- ##### 执行存储过程的对象

CallableStatement prepareCall(sql)

#### 事务管理

- ##### MySQL 事务管理

  - 开启事务：BEGIN; / START TRANSACTION;
  - 提交事务：COMMIT;
  - 回滚事务：ROLLBACK;

MySQL默认自动提交事务

- ##### JDBC 事务管理：

  Connection接口中定义了3个对应的方法

  开启事务：setAutoCommit(boolean autoCommit): true为自动提交事务；false为手动提交事务，即为开启事务
  提交事务：commit()
  回滚事务：rollback()

# Statement

- Statement作用：
  1. 执行SQL语句

##### int executeUpdate(sql): 执行DML、DDL语句

返回值：(1) DML语句影响的行数 (2) DDL语句执行后，执行成功也可能返回 0

##### ResultSet executeQuery(sql): 执行DQL语句

返回值：ResultSet 结果集对象

# ResultSet(结果集对象)

- **ResultSet(结果集对象)作用**:
  1. 封装了DQL查询语句的结果  
  
     ```
     `ResultSet stmt.executeQuery(sql):` 执行DQL语句，返回ResultSet对象  
     ```
  
- **获取查询结果**

```
boolean next(): 
(1) 将光标从当前位置向前移动一行  
(2) 判断当前行是否为有效行  

- **返回值**:  
  - true: 有效行，当前行有数据  
  - false: 无效行，当前行没有数据  

---
```

```
xxx getXxx(参数): 获取数据  

- xxx: 数据类型; 如: int getInt(参数); String getString(参数)  
- **参数**:  
  - int: 列的编号，从1开始  
  - String: 列的名称
```

#### 使用步骤：

1. 游标向下移动一行，并判断该行是否有数据：next()
2. 获取数据：getXxx(参数)

```java
//循环判断游标是否是最后一行末尾
while(rs.next()){
//获取数据
rs.getXxx(参数);
}
```

# **PreparedStatement**

##### 预防sql注入问题

**PreparedStatement**预防SQL 注入的原理：将敏感字符进行转义

## SQL注入

**SQL注入**是一种通过**将恶意SQL代码拼接到正常SQL语句中**，从而破坏SQL语句原有结构，达到绕过验证、窃取数据或破坏数据库的攻击方式。

### 经典案例

假设登录验证的SQL语句为：

```
SELECT * FROM users WHERE username = '输入的用户名' AND password = '输入的密码';
```

如果攻击者输入：

- **用户名**：`admin' --`
- **密码**：随便输

拼接后的SQL变成：

```
SELECT * FROM users WHERE username = 'admin' -- ' AND password = '随便';
```

> `--` 是SQL中的注释符，后面的条件全被注释掉，**无需密码即可登录**。

如果输入：

- **用户名**：`admin' OR '1'='1`

拼接后：

```
SELECT * FROM users WHERE username = 'admin' OR '1'='1' AND password = '...';
```

> `OR '1'='1'` 永远为真，**直接绕过认证**。

#### 使用 PreparedStatement防御

```java
String sql = "SELECT * FROM users WHERE username = ? AND password = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, userInput);
pstmt.setString(2, passInput);
ResultSet rs = pstmt.executeQuery();
```

> `?` 占位符传入的值会**自动转义**，不参与SQL结构解析，从根本上杜绝注入。

![image-20260827180027361](C:\Users\ThinkPad\AppData\Roaming\Typora\typora-user-images\image-20260827180027361.png)

# 数据库连接池

![image-20260827180837437](C:\Users\ThinkPad\AppData\Roaming\Typora\typora-user-images\image-20260827180837437.png)

#### 连接池 = 一个存放数据库连接的缓存容器（集合）+ 一套管理这些连接生命周期的逻辑

##### 连接池的实现

- **标准接口**: DataSource
  - 官方(SUN)提供的数据库连接池标准接口，由第三方组织实现此接口。
  - 功能: 获取连接

# Driud使用步骤

1. 导入jar包 druid-1.1.12.jar  
2. 定义配置文件  
3. 加载配置文件  
4. 获取数据库连接池对象  
5. 获取连接

# Maven

- 提供了一套标准化的项目结构  
- 提供了一套标准化的构建流程（编译、测试、打包、发布……）  
- 提供了一套依赖管理机制

### maven仓库

* 本地仓库：计算机目录
* 中央仓库：全球唯一
* 远程仓库（私服）：公司自己维护的仓库

### Maven坐标

**什么是坐标？**

- Maven 中的坐标是资源的唯一标识
- 使用坐标来定义项目或引入项目中需要的依赖

**Maven 坐标主要组成**

- `groupId`：定义当前Maven项目隶属组织名称（通常是域名反写，例如：com.itheima）
- `artifactId`：定义当前Maven项目名称（通常是模块名称，例如 order-service、goods-service）
- `version`：定义当前项目版本号

# Mybatis

##### 一款持久层框架，用于简化JDBC开发

JavaEE三层架构：

* 表现层：也称为 **Web 层** 或 **用户界面层**，是直接与用户交互的层次。
* 业务层：也称为 **Service 层** 或 **应用层**，是系统的核心，负责处理具体的业务逻辑
* 持久层：也称为 **DAO 层（Data Access Object）** 或 **数据访问层**，负责与数据库进行交互。

硬编码问题：就是字符串，连接地址什么的写进代码里

解决方式：把这些东西都写到配置文件里，用到的时候读取配置文件