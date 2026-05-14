# SQL

## 1. SQL 简介

SQL（Structured Query Language，结构化查询语言）是用于管理关系型数据库的标准语言，由 IBM 在 1970 年代开发。

### 主要数据库系统对比

| 数据库       | 特点                        | 适用场景           |
|------------|-----------------------------|--------------------|
| MySQL      | 开源、高性能、使用广泛        | Web 应用、中小型系统 |
| PostgreSQL | 开源、功能强大、支持 JSON    | 复杂查询、企业级应用 |
| SQLite     | 轻量级、嵌入式、无服务端      | 移动应用、本地存储   |
| SQL Server | 微软出品、企业级              | Windows 生态系统    |
| Oracle     | 功能完备、性能极强            | 大型企业、金融系统   |

### SQL 语言分类

```
SQL
├── DDL（Data Definition Language）  数据定义语言  — CREATE / ALTER / DROP
├── DML（Data Manipulation Language）数据操作语言  — INSERT / UPDATE / DELETE
├── DQL（Data Query Language）       数据查询语言  — SELECT
└── DCL（Data Control Language）     数据控制语言  — GRANT / REVOKE
```

---

## 2. 数据库基础概念

### 核心术语

| 术语   | 说明                                       |
|------|--------------------------------------------|
| 数据库  | 按结构组织的数据集合                         |
| 表    | 数据以行列形式存储的二维结构                   |
| 行/记录 | 表中的一条数据                              |
| 列/字段 | 表中某一类属性                              |
| 主键   | 唯一标识一条记录的字段，不可为 NULL           |
| 外键   | 引用另一张表主键的字段，用于建立表间关系       |
| 索引   | 加速查询的数据结构，类似书的目录              |
| 视图   | 基于查询结果的虚拟表                         |
| 事务   | 一组原子性操作，要么全部成功，要么全部回滚      |

---

## 3. 数据类型

### 数值类型

```sql
INT           -- 整数，4字节，范围 -2^31 ~ 2^31-1
BIGINT        -- 大整数，8字节
SMALLINT      -- 小整数，2字节
TINYINT       -- 极小整数，1字节（0~255）
DECIMAL(p,s)  -- 精确小数，p位总长，s位小数，适合金额
FLOAT         -- 单精度浮点
DOUBLE        -- 双精度浮点
```

### 字符串类型

```sql
CHAR(n)       -- 固定长度字符串，不足补空格，最大 255
VARCHAR(n)    -- 可变长度字符串，最大 65535（实际受行长限制）
TEXT          -- 长文本，最大 65535 字节
LONGTEXT      -- 超长文本，最大 4GB
ENUM('a','b') -- 枚举类型，只能取预定义值之一
```

### 日期时间类型

```sql
DATE          -- 日期，格式 YYYY-MM-DD
TIME          -- 时间，格式 HH:MM:SS
DATETIME      -- 日期时间，格式 YYYY-MM-DD HH:MM:SS
TIMESTAMP     -- 时间戳，自动记录修改时间，受时区影响
YEAR          -- 年份
```

### 其他类型

```sql
BOOLEAN       -- 布尔值（MySQL 中实为 TINYINT(1)）
JSON          -- JSON 格式数据（MySQL 5.7+，PostgreSQL 原生支持）
BLOB          -- 二进制大对象，存储图片/文件等
```

---

## 4. DDL — 数据定义语言

### 4.1 创建数据库

```sql
-- 创建数据库
CREATE DATABASE shop;

-- 创建时指定字符集（推荐 utf8mb4 以支持 emoji）
CREATE DATABASE shop
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

-- 如果不存在则创建
CREATE DATABASE IF NOT EXISTS shop;

-- 查看所有数据库
SHOW DATABASES;

-- 切换数据库
USE shop;

-- 删除数据库（谨慎！）
DROP DATABASE shop;
```

### 4.2 创建表

```sql
CREATE TABLE users (
    id          INT             NOT NULL AUTO_INCREMENT,
    username    VARCHAR(50)     NOT NULL,
    email       VARCHAR(100)    NOT NULL,
    password    VARCHAR(255)    NOT NULL,
    age         TINYINT         UNSIGNED,
    balance     DECIMAL(10, 2)  DEFAULT 0.00,
    status      ENUM('active', 'inactive', 'banned') DEFAULT 'active',
    created_at  DATETIME        DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME        DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_email (email),
    INDEX idx_username (username)
);
```

### 4.3 修改表结构

```sql
-- 添加列
ALTER TABLE users ADD COLUMN phone VARCHAR(20) AFTER email;

-- 删除列
ALTER TABLE users DROP COLUMN phone;

-- 修改列定义（改类型/约束）
ALTER TABLE users MODIFY COLUMN age SMALLINT UNSIGNED;

-- 重命名列
ALTER TABLE users CHANGE COLUMN username user_name VARCHAR(50) NOT NULL;

-- 添加索引
ALTER TABLE users ADD INDEX idx_status (status);

-- 删除索引
ALTER TABLE users DROP INDEX idx_status;

-- 重命名表
ALTER TABLE users RENAME TO members;
-- 或
RENAME TABLE members TO users;
```

### 4.4 删除与清空表

```sql
-- 删除表（结构和数据一起删除）
DROP TABLE users;

-- 安全删除
DROP TABLE IF EXISTS users;

-- 清空表数据（保留结构，重置自增 ID，DDL 操作，不可回滚）
TRUNCATE TABLE users;

-- 清空数据（DML 操作，可回滚，自增不重置）
DELETE FROM users;
```

---

## 5. DML — 数据操作语言

### 5.1 INSERT 插入数据

```sql
-- 插入单行（推荐明确列名）
INSERT INTO users (username, email, password, age)
VALUES ('alice', 'alice@example.com', 'hashed_pwd', 25);

-- 插入多行
INSERT INTO users (username, email, password)
VALUES
    ('bob',   'bob@example.com',   'pwd1'),
    ('carol', 'carol@example.com', 'pwd2'),
    ('dave',  'dave@example.com',  'pwd3');

-- 从另一张表查询并插入
INSERT INTO users_backup (username, email)
SELECT username, email FROM users WHERE status = 'active';

-- 存在则更新，不存在则插入（MySQL）
INSERT INTO users (id, username, email)
VALUES (1, 'alice_new', 'alice_new@example.com')
ON DUPLICATE KEY UPDATE username = VALUES(username), email = VALUES(email);
```

### 5.2 UPDATE 更新数据

```sql
-- 更新单列
UPDATE users SET status = 'inactive' WHERE id = 1;

-- 更新多列
UPDATE users
SET username = 'alice2', email = 'alice2@example.com', updated_at = NOW()
WHERE id = 1;

-- 批量更新（加 LIMIT 防止误操作）
UPDATE users SET status = 'inactive'
WHERE created_at < '2024-01-01'
LIMIT 100;

-- 基于另一张表的值更新（JOIN UPDATE）
UPDATE users u
JOIN orders o ON u.id = o.user_id
SET u.balance = u.balance - o.amount
WHERE o.status = 'paid';
```

### 5.3 DELETE 删除数据

```sql
-- 删除指定行
DELETE FROM users WHERE id = 1;

-- 删除多行
DELETE FROM users WHERE status = 'banned';

-- 限制删除数量（防止误删大量数据）
DELETE FROM users WHERE status = 'inactive' LIMIT 10;

-- 多表联合删除（删除 users 表中有订单未支付的用户）
DELETE u FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'unpaid';
```

---

## 6. DQL — 数据查询语言

### 6.1 SELECT 基础

```sql
-- 查询所有列
SELECT * FROM users;

-- 查询指定列
SELECT id, username, email FROM users;

-- 列别名
SELECT username AS name, email AS contact FROM users;

-- 去重
SELECT DISTINCT status FROM users;

-- 计算列
SELECT username, balance * 1.1 AS adjusted_balance FROM users;

-- 字符串拼接
SELECT CONCAT(username, ' <', email, '>') AS display_name FROM users;
```

### 6.2 WHERE 条件过滤

```sql
-- 比较运算符
SELECT * FROM users WHERE age > 18;
SELECT * FROM users WHERE age BETWEEN 18 AND 30;   -- 包含边界
SELECT * FROM users WHERE age NOT BETWEEN 18 AND 30;

-- 字符串匹配
SELECT * FROM users WHERE username LIKE 'a%';      -- 以 a 开头
SELECT * FROM users WHERE email LIKE '%@gmail.com';-- 以 @gmail.com 结尾
SELECT * FROM users WHERE username LIKE '_lice';   -- _ 匹配单个字符

-- 集合匹配
SELECT * FROM users WHERE status IN ('active', 'inactive');
SELECT * FROM users WHERE id NOT IN (1, 2, 3);

-- NULL 判断（不能用 = NULL）
SELECT * FROM users WHERE age IS NULL;
SELECT * FROM users WHERE age IS NOT NULL;

-- 逻辑运算
SELECT * FROM users WHERE age > 18 AND status = 'active';
SELECT * FROM users WHERE status = 'banned' OR balance < 0;
SELECT * FROM users WHERE NOT (status = 'active');
```

### 6.3 ORDER BY 排序

```sql
-- 升序（默认）
SELECT * FROM users ORDER BY age ASC;

-- 降序
SELECT * FROM users ORDER BY created_at DESC;

-- 多列排序：先按 status 升序，status 相同时按 created_at 降序
SELECT * FROM users ORDER BY status ASC, created_at DESC;

-- NULL 值排序（MySQL 中 NULL 排在最前，可用 IS NULL 控制）
SELECT * FROM users ORDER BY age IS NULL, age ASC;
```

### 6.4 LIMIT 分页

```sql
-- 取前 10 条
SELECT * FROM users LIMIT 10;

-- 分页：跳过前 20 条，取第 21~30 条（第 3 页，每页 10 条）
SELECT * FROM users LIMIT 10 OFFSET 20;
-- 等价写法（MySQL 特有，更简洁）
SELECT * FROM users LIMIT 20, 10;

-- 通用分页公式
-- LIMIT {pageSize} OFFSET {(pageNum - 1) * pageSize}
```

### 6.5 常用函数

#### 字符串函数

```sql
SELECT LENGTH('hello');              -- 5（字节数）
SELECT CHAR_LENGTH('hello');         -- 5（字符数，多字节字符安全）
SELECT UPPER('hello');               -- HELLO
SELECT LOWER('HELLO');               -- hello
SELECT TRIM('  hello  ');            -- 'hello'
SELECT LTRIM('  hello');             -- 'hello'
SELECT RTRIM('hello  ');             -- 'hello'
SELECT SUBSTRING('hello', 2, 3);     -- 'ell'（从第2位取3个字符）
SELECT REPLACE('hello world', 'world', 'SQL'); -- 'hello SQL'
SELECT CONCAT('a', 'b', 'c');        -- 'abc'
SELECT CONCAT_WS(',', 'a', 'b', 'c');-- 'a,b,c'（带分隔符）
SELECT INSTR('hello', 'ell');        -- 2（位置）
SELECT LPAD('5', 3, '0');            -- '005'
SELECT RPAD('5', 3, '0');            -- '500'
```

#### 数值函数

```sql
SELECT ABS(-5);        -- 5
SELECT CEIL(4.1);      -- 5（向上取整）
SELECT FLOOR(4.9);     -- 4（向下取整）
SELECT ROUND(4.567, 2);-- 4.57（四舍五入保留2位小数）
SELECT MOD(10, 3);     -- 1（取余）
SELECT POWER(2, 10);   -- 1024
SELECT SQRT(16);       -- 4
SELECT RAND();         -- 0~1 之间随机数
```

#### 日期函数

```sql
SELECT NOW();                        -- 当前日期时间
SELECT CURDATE();                    -- 当前日期
SELECT CURTIME();                    -- 当前时间
SELECT YEAR(NOW());                  -- 年
SELECT MONTH(NOW());                 -- 月
SELECT DAY(NOW());                   -- 日
SELECT HOUR(NOW());                  -- 时
SELECT DAYOFWEEK(NOW());             -- 星期几（1=周日）
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s'); -- 格式化
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);  -- 7天后
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);-- 1个月前
SELECT DATEDIFF('2026-12-31', '2026-01-01'); -- 相差天数 364
SELECT TIMESTAMPDIFF(MONTH, '2026-01-01', '2026-05-01'); -- 相差月数 4
```

#### 条件函数

```sql
-- IF(条件, 真值, 假值)
SELECT username, IF(age >= 18, '成年', '未成年') AS is_adult FROM users;

-- IFNULL(值, 默认值)
SELECT IFNULL(age, 0) AS safe_age FROM users;

-- COALESCE：返回第一个非 NULL 值
SELECT COALESCE(phone, email, '无联系方式') AS contact FROM users;

-- CASE WHEN
SELECT username,
    CASE
        WHEN age < 18  THEN '未成年'
        WHEN age < 60  THEN '成年'
        ELSE '老年'
    END AS age_group
FROM users;

-- CASE 值匹配（类似 switch）
SELECT username,
    CASE status
        WHEN 'active'   THEN '正常'
        WHEN 'inactive' THEN '停用'
        WHEN 'banned'   THEN '封禁'
        ELSE '未知'
    END AS status_text
FROM users;
```

---

## 7. DCL — 数据控制语言

```sql
-- 创建用户
CREATE USER 'dev'@'localhost' IDENTIFIED BY 'password123';

-- 授权
GRANT SELECT, INSERT, UPDATE ON shop.* TO 'dev'@'localhost';

-- 授予所有权限
GRANT ALL PRIVILEGES ON shop.* TO 'dev'@'localhost';

-- 刷新权限
FLUSH PRIVILEGES;

-- 查看用户权限
SHOW GRANTS FOR 'dev'@'localhost';

-- 撤销权限
REVOKE INSERT ON shop.* FROM 'dev'@'localhost';

-- 删除用户
DROP USER 'dev'@'localhost';
```

---

## 8. 约束

### 约束类型

```sql
CREATE TABLE orders (
    id          INT         NOT NULL AUTO_INCREMENT,
    user_id     INT         NOT NULL,
    product_id  INT         NOT NULL,
    quantity    INT         NOT NULL DEFAULT 1,
    amount      DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',

    PRIMARY KEY (id),                              -- 主键约束
    UNIQUE KEY uk_user_product (user_id, product_id), -- 联合唯一约束
    CONSTRAINT fk_user
        FOREIGN KEY (user_id) REFERENCES users(id)
        ON DELETE CASCADE                          -- 父记录删除时级联删除子记录
        ON UPDATE CASCADE,                         -- 父记录更新时级联更新
    CONSTRAINT chk_quantity CHECK (quantity > 0),  -- 检查约束（MySQL 8.0+）
    INDEX idx_status (status)
);
```

### 外键行为说明

| 行为         | 说明                           |
|------------|-------------------------------|
| RESTRICT   | 父记录有子记录时拒绝删除/更新（默认）|
| CASCADE    | 父记录删除/更新时，子记录跟着操作   |
| SET NULL   | 父记录删除/更新时，子外键列置 NULL  |
| NO ACTION  | 与 RESTRICT 类似                |

---

## 9. 索引

### 索引类型

```sql
-- 普通索引
CREATE INDEX idx_username ON users(username);

-- 唯一索引
CREATE UNIQUE INDEX uk_email ON users(email);

-- 联合索引（遵循最左前缀原则）
CREATE INDEX idx_status_created ON users(status, created_at);

-- 全文索引（适合大文本搜索）
CREATE FULLTEXT INDEX ft_content ON articles(title, content);
-- 使用全文索引
SELECT * FROM articles WHERE MATCH(title, content) AGAINST('SQL 教程' IN BOOLEAN MODE);

-- 前缀索引（节省索引空间）
CREATE INDEX idx_email_prefix ON users(email(20));

-- 查看索引
SHOW INDEX FROM users;

-- 删除索引
DROP INDEX idx_username ON users;
ALTER TABLE users DROP INDEX idx_username;
```

### 最左前缀原则

联合索引 `(a, b, c)` 等价于同时建立了：
- `(a)`
- `(a, b)`
- `(a, b, c)`

```sql
-- 以下查询可以使用索引
SELECT * FROM t WHERE a = 1;
SELECT * FROM t WHERE a = 1 AND b = 2;
SELECT * FROM t WHERE a = 1 AND b = 2 AND c = 3;

-- 以下查询无法使用索引（跳过了 a）
SELECT * FROM t WHERE b = 2;
SELECT * FROM t WHERE b = 2 AND c = 3;
```

### 索引失效场景

```sql
-- 对列使用函数（索引失效）
SELECT * FROM users WHERE YEAR(created_at) = 2026;
-- 改写为范围查询（使用索引）
SELECT * FROM users WHERE created_at BETWEEN '2026-01-01' AND '2026-12-31';

-- LIKE 以通配符开头（失效）
SELECT * FROM users WHERE username LIKE '%alice';
-- LIKE 以通配符结尾（可用索引）
SELECT * FROM users WHERE username LIKE 'alice%';

-- 隐式类型转换（字段是 VARCHAR，传入整数，失效）
SELECT * FROM users WHERE email = 12345;

-- 使用 OR 连接不同列（失效，除非两列都有索引）
SELECT * FROM users WHERE id = 1 OR username = 'alice';

-- != 或 NOT IN 通常无法使用索引
SELECT * FROM users WHERE status != 'active';
```

---

## 10. JOIN 连接查询

### JOIN 类型

```sql
-- 准备数据
-- users: id, username
-- orders: id, user_id, amount

-- INNER JOIN（内连接）：只返回两表都匹配的行
SELECT u.username, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN（左外连接）：返回左表所有行，右表无匹配则 NULL
SELECT u.username, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT JOIN（右外连接）：返回右表所有行，左表无匹配则 NULL
SELECT u.username, o.amount
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN（全外连接，MySQL 不直接支持，用 UNION 模拟）
SELECT u.username, o.amount
FROM users u LEFT JOIN orders o ON u.id = o.user_id
UNION
SELECT u.username, o.amount
FROM users u RIGHT JOIN orders o ON u.id = o.user_id;

-- CROSS JOIN（笛卡尔积，慎用）
SELECT u.username, p.name
FROM users u CROSS JOIN products p;

-- 自连接（表和自身连接，常用于层级数据）
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### 多表 JOIN

```sql
SELECT
    u.username,
    o.id AS order_id,
    p.name AS product_name,
    oi.quantity,
    oi.price
FROM users u
JOIN orders o        ON u.id = o.user_id
JOIN order_items oi  ON o.id = oi.order_id
JOIN products p      ON oi.product_id = p.id
WHERE o.status = 'paid'
ORDER BY o.created_at DESC;
```

---

## 11. 子查询

### 标量子查询（返回单个值）

```sql
-- 查询余额高于平均值的用户
SELECT username, balance
FROM users
WHERE balance > (SELECT AVG(balance) FROM users);
```

### 列子查询（返回一列多行）

```sql
-- 查询有订单的用户
SELECT username FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- 查询没有订单的用户
SELECT username FROM users
WHERE id NOT IN (SELECT DISTINCT user_id FROM orders WHERE user_id IS NOT NULL);
```

### 行子查询（返回一行多列）

```sql
SELECT * FROM orders
WHERE (user_id, amount) = (SELECT user_id, MAX(amount) FROM orders WHERE user_id = 1);
```

### 表子查询（返回多行多列，用于 FROM）

```sql
-- 每个用户的订单统计
SELECT u.username, order_stats.order_count, order_stats.total_amount
FROM users u
JOIN (
    SELECT user_id, COUNT(*) AS order_count, SUM(amount) AS total_amount
    FROM orders
    GROUP BY user_id
) AS order_stats ON u.id = order_stats.user_id;
```

### EXISTS / NOT EXISTS

```sql
-- 查询有订单的用户（EXISTS 通常比 IN 性能好，尤其是大数据量）
SELECT username FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- 查询没有订单的用户
SELECT username FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

### CTE（公共表表达式，WITH 语句）

```sql
-- 简单 CTE
WITH active_users AS (
    SELECT id, username FROM users WHERE status = 'active'
)
SELECT u.username, COUNT(o.id) AS order_count
FROM active_users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;

-- 多个 CTE
WITH
    monthly_sales AS (
        SELECT DATE_FORMAT(created_at, '%Y-%m') AS month, SUM(amount) AS total
        FROM orders WHERE status = 'paid'
        GROUP BY month
    ),
    avg_sales AS (
        SELECT AVG(total) AS avg_monthly FROM monthly_sales
    )
SELECT ms.month, ms.total,
       ROUND(ms.total / a.avg_monthly * 100, 1) AS pct_of_avg
FROM monthly_sales ms, avg_sales a
ORDER BY ms.month;

-- 递归 CTE（处理层级数据，如组织架构）
WITH RECURSIVE org_tree AS (
    -- 基础查询：顶层员工
    SELECT id, name, manager_id, 0 AS level
    FROM employees WHERE manager_id IS NULL

    UNION ALL

    -- 递归查询：向下扩展
    SELECT e.id, e.name, e.manager_id, ot.level + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT LPAD('', level * 2, ' ') || name AS hierarchy
FROM org_tree
ORDER BY level, name;
```

---

## 12. 聚合函数与分组

### 聚合函数

```sql
SELECT
    COUNT(*)           AS total_rows,      -- 总行数（含NULL）
    COUNT(age)         AS age_not_null,    -- 非NULL的行数
    COUNT(DISTINCT status) AS unique_status,-- 去重计数
    SUM(balance)       AS total_balance,
    AVG(balance)       AS avg_balance,
    MAX(balance)       AS max_balance,
    MIN(balance)       AS min_balance,
    GROUP_CONCAT(username ORDER BY username SEPARATOR ', ') AS names -- 字符串聚合
FROM users;
```

### GROUP BY 分组

```sql
-- 按状态统计用户数和平均余额
SELECT status, COUNT(*) AS cnt, AVG(balance) AS avg_balance
FROM users
GROUP BY status;

-- 多列分组
SELECT status, YEAR(created_at) AS year, COUNT(*) AS cnt
FROM users
GROUP BY status, year
ORDER BY year, status;
```

### HAVING 分组后过滤

```sql
-- WHERE 在分组前过滤原始数据，HAVING 在分组后过滤聚合结果
-- 找出订单数超过 5 的用户
SELECT user_id, COUNT(*) AS order_count, SUM(amount) AS total
FROM orders
WHERE status = 'paid'          -- 先过滤：只统计已支付订单
GROUP BY user_id
HAVING order_count > 5         -- 再过滤：只返回订单数 > 5 的
ORDER BY total DESC;
```

### ROLLUP（小计/合计）

```sql
SELECT status, YEAR(created_at) AS year, COUNT(*) AS cnt
FROM users
GROUP BY status, year WITH ROLLUP;
-- ROLLUP 会额外生成每个 status 的小计行，以及所有数据的总计行
```

---

## 13. 窗口函数

窗口函数在不折叠行的情况下，对每一行基于一个"窗口"（分区 + 排序）进行计算，是分析类查询的利器。

### 语法

```sql
函数名() OVER (
    PARTITION BY 分区列        -- 可选：按哪列分区，类似 GROUP BY
    ORDER BY 排序列            -- 可选：窗口内排序
    ROWS BETWEEN ... AND ...   -- 可选：窗口帧范围
)
```

### 排名函数

```sql
SELECT
    username,
    balance,
    ROW_NUMBER()   OVER (ORDER BY balance DESC) AS row_num,  -- 唯一序号，无并列
    RANK()         OVER (ORDER BY balance DESC) AS rnk,      -- 并列时跳号（1,1,3）
    DENSE_RANK()   OVER (ORDER BY balance DESC) AS dense_rnk,-- 并列时不跳号（1,1,2）
    NTILE(4)       OVER (ORDER BY balance DESC) AS quartile   -- 分成4组
FROM users;
```

### 分区排名

```sql
-- 每个状态组内的余额排名
SELECT
    username, status, balance,
    RANK() OVER (PARTITION BY status ORDER BY balance DESC) AS rank_in_group
FROM users;
```

### 偏移函数

```sql
-- 查看每个用户相比前一名的余额差
SELECT
    username,
    balance,
    LAG(balance, 1)  OVER (ORDER BY balance DESC) AS prev_balance,
    LEAD(balance, 1) OVER (ORDER BY balance DESC) AS next_balance,
    balance - LAG(balance, 1) OVER (ORDER BY balance DESC) AS diff
FROM users;
```

### 聚合窗口函数

```sql
-- 累计求和（running total）
SELECT
    created_at,
    amount,
    SUM(amount) OVER (ORDER BY created_at) AS running_total
FROM orders WHERE status = 'paid';

-- 移动平均（前后各 2 行）
SELECT
    created_at,
    amount,
    AVG(amount) OVER (
        ORDER BY created_at
        ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
    ) AS moving_avg
FROM orders;

-- 分组内占比
SELECT
    status,
    username,
    balance,
    ROUND(balance / SUM(balance) OVER (PARTITION BY status) * 100, 2) AS pct
FROM users;
```

---

## 14. 视图

```sql
-- 创建视图
CREATE VIEW active_user_summary AS
SELECT
    u.id,
    u.username,
    u.email,
    COUNT(o.id)   AS order_count,
    SUM(o.amount) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id, u.username, u.email;

-- 使用视图（像普通表一样查询）
SELECT * FROM active_user_summary WHERE total_spent > 1000;

-- 修改视图
CREATE OR REPLACE VIEW active_user_summary AS
SELECT id, username FROM users WHERE status = 'active';

-- 删除视图
DROP VIEW active_user_summary;
```

**视图的优缺点：**

| 优点                         | 缺点                           |
|------------------------------|-------------------------------|
| 简化复杂查询                  | 不存储数据，查询时实时计算       |
| 数据安全（隐藏敏感字段）        | 复杂视图性能差                  |
| 逻辑独立（修改底层表不影响视图） | 大多数视图不可直接 INSERT/UPDATE |

---

## 15. 存储过程与函数

### 存储过程

```sql
DELIMITER $$

CREATE PROCEDURE get_user_orders(IN p_user_id INT, IN p_status VARCHAR(20))
BEGIN
    SELECT o.id, o.amount, o.created_at
    FROM orders o
    WHERE o.user_id = p_user_id
      AND (p_status IS NULL OR o.status = p_status)
    ORDER BY o.created_at DESC;
END$$

DELIMITER ;

-- 调用存储过程
CALL get_user_orders(1, 'paid');
CALL get_user_orders(1, NULL);  -- NULL 表示不过滤状态
```

### 带 OUT 参数的存储过程

```sql
DELIMITER $$

CREATE PROCEDURE transfer_balance(
    IN  p_from_id  INT,
    IN  p_to_id    INT,
    IN  p_amount   DECIMAL(10,2),
    OUT p_result   VARCHAR(50)
)
BEGIN
    DECLARE v_balance DECIMAL(10,2);

    START TRANSACTION;

    SELECT balance INTO v_balance FROM users WHERE id = p_from_id FOR UPDATE;

    IF v_balance < p_amount THEN
        ROLLBACK;
        SET p_result = '余额不足';
    ELSE
        UPDATE users SET balance = balance - p_amount WHERE id = p_from_id;
        UPDATE users SET balance = balance + p_amount WHERE id = p_to_id;
        COMMIT;
        SET p_result = '转账成功';
    END IF;
END$$

DELIMITER ;

-- 调用
CALL transfer_balance(1, 2, 100.00, @result);
SELECT @result;
```

### 自定义函数

```sql
DELIMITER $$

CREATE FUNCTION age_group(age INT)
RETURNS VARCHAR(10)
DETERMINISTIC
BEGIN
    DECLARE result VARCHAR(10);
    IF age < 18 THEN
        SET result = '未成年';
    ELSEIF age < 60 THEN
        SET result = '成年';
    ELSE
        SET result = '老年';
    END IF;
    RETURN result;
END$$

DELIMITER ;

-- 使用函数
SELECT username, age, age_group(age) AS group_name FROM users;
```

---

## 16. 事务

### ACID 特性

| 特性         | 说明                                         |
|------------|----------------------------------------------|
| 原子性 (A)  | 事务中所有操作要么全部成功，要么全部回滚          |
| 一致性 (C)  | 事务前后数据库处于合法状态                      |
| 隔离性 (I)  | 并发事务之间互不干扰                            |
| 持久性 (D)  | 事务提交后，数据永久保存，即使系统崩溃           |

### 基本事务操作

```sql
-- 开启事务
START TRANSACTION;
-- 或
BEGIN;

-- 执行操作
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

-- 提交（持久化所有操作）
COMMIT;

-- 回滚（撤销所有操作）
ROLLBACK;

-- 保存点（可以回滚到中间状态）
START TRANSACTION;
  UPDATE users SET balance = balance - 100 WHERE id = 1;
  SAVEPOINT sp1;
  UPDATE users SET balance = balance - 200 WHERE id = 2;
  ROLLBACK TO SAVEPOINT sp1;  -- 只回滚到 sp1，第一个 UPDATE 保留
COMMIT;
```

### 隔离级别

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置隔离级别（SESSION 只影响当前连接）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

| 隔离级别           | 脏读 | 不可重复读 | 幻读 |
|-----------------|:--:|:------:|:--:|
| READ UNCOMMITTED | ✓  |   ✓    |  ✓ |
| READ COMMITTED   | ✗  |   ✓    |  ✓ |
| REPEATABLE READ  | ✗  |   ✗    |  ✓ |
| SERIALIZABLE     | ✗  |   ✗    |  ✗ |

> MySQL InnoDB 默认 `REPEATABLE READ`，通过 MVCC 和间隙锁在很大程度上解决了幻读问题。

---

## 17. 性能优化

### 使用 EXPLAIN 分析查询

```sql
EXPLAIN SELECT u.username, COUNT(o.id) AS cnt
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id;
```

**EXPLAIN 关键字段：**

| 字段          | 说明                                                      |
|-------------|----------------------------------------------------------|
| type        | 访问类型：system > const > eq_ref > ref > range > index > ALL |
| key         | 实际使用的索引                                             |
| key_len     | 索引使用的字节数                                           |
| rows        | 预估扫描行数                                               |
| Extra       | Using filesort / Using temporary 表示需要优化              |

### 优化技巧

```sql
-- 1. 避免 SELECT *，只取需要的列
SELECT id, username FROM users WHERE status = 'active';

-- 2. 分页深翻优化（用 id 条件代替大 OFFSET）
-- 慢（OFFSET 100000 会扫描大量数据）
SELECT * FROM orders LIMIT 100000, 10;
-- 快（利用主键索引）
SELECT * FROM orders WHERE id > 100000 LIMIT 10;

-- 3. 用 EXISTS 替代 IN（大数据集时）
-- 慢
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE amount > 1000);
-- 快
SELECT * FROM users u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.amount > 1000
);

-- 4. 批量插入替代逐条插入
INSERT INTO logs (user_id, action, created_at) VALUES
    (1, 'login',  NOW()),
    (2, 'logout', NOW()),
    (3, 'login',  NOW());

-- 5. 避免在 WHERE 中对索引列使用函数
-- 慢（索引失效）
SELECT * FROM orders WHERE DATE(created_at) = '2026-05-01';
-- 快（范围扫描，使用索引）
SELECT * FROM orders
WHERE created_at >= '2026-05-01 00:00:00'
  AND created_at <  '2026-05-02 00:00:00';
```

---

## 18. 常见面试题

### 题 1：查找重复数据

```sql
-- 找出 email 重复的记录
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING cnt > 1;

-- 删除重复保留 id 最小的一条
DELETE FROM users
WHERE id NOT IN (
    SELECT MIN(id) FROM users GROUP BY email
);
```

### 题 2：查询每个部门薪资最高的员工

```sql
-- 方法 1：子查询
SELECT e.*
FROM employees e
WHERE salary = (
    SELECT MAX(salary) FROM employees WHERE dept_id = e.dept_id
);

-- 方法 2：窗口函数（推荐，更简洁）
SELECT *
FROM (
    SELECT *, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 1;
```

### 题 3：连续登录天数

```sql
-- 找出每个用户最长连续登录天数
WITH daily AS (
    SELECT user_id, DATE(login_time) AS login_date
    FROM login_logs
    GROUP BY user_id, login_date
),
grouped AS (
    SELECT
        user_id, login_date,
        DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ) DAY) AS grp
    FROM daily
)
SELECT user_id, MAX(cnt) AS max_consecutive
FROM (
    SELECT user_id, grp, COUNT(*) AS cnt
    FROM grouped
    GROUP BY user_id, grp
) t
GROUP BY user_id;
```

### 题 4：行列转换（PIVOT）

```sql
-- 原始数据：user_id, subject, score
-- 目标：每人一行，科目为列

SELECT user_id,
    MAX(CASE WHEN subject = '语文' THEN score END) AS chinese,
    MAX(CASE WHEN subject = '数学' THEN score END) AS math,
    MAX(CASE WHEN subject = '英语' THEN score END) AS english
FROM scores
GROUP BY user_id;
```

### 题 5：同比/环比计算

```sql
WITH monthly AS (
    SELECT
        DATE_FORMAT(created_at, '%Y-%m') AS month,
        SUM(amount) AS revenue
    FROM orders WHERE status = 'paid'
    GROUP BY month
)
SELECT
    month,
    revenue,
    LAG(revenue, 1)  OVER (ORDER BY month) AS last_month,
    LAG(revenue, 12) OVER (ORDER BY month) AS same_month_last_year,
    ROUND((revenue - LAG(revenue, 1) OVER (ORDER BY month)) /
          LAG(revenue, 1) OVER (ORDER BY month) * 100, 2) AS mom_growth_pct
FROM monthly;
```

---

## 快速参考卡

```
SELECT  列           -- 选择哪些列
FROM    表            -- 数据来源
JOIN    表 ON 条件    -- 关联其他表
WHERE   条件          -- 过滤原始行（在聚合前）
GROUP BY 列          -- 分组聚合
HAVING  条件          -- 过滤聚合后的结果
ORDER BY 列 [ASC|DESC]-- 排序
LIMIT   n OFFSET m   -- 分页取数

执行顺序：FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```
