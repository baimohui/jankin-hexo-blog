---
title: MySQL 详解
categories: 
- 后端
tags:
- 数据库
- MySQL
- SQL
- 关系型数据库
---

## 一、连接与基本操作

```bash
mysql -u root -p                           # 连接
SHOW DATABASES;                            # 查看数据库
CREATE DATABASE myapp CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE myapp;                                 # 使用数据库
SELECT DATABASE();                         # 查看当前数据库
```

## 二、表操作

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  age INT DEFAULT 0,
  status TINYINT DEFAULT 1 COMMENT '1:active 0:inactive',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DESCRIBE users;                            -- 查看表结构
SHOW CREATE TABLE users;                   -- 查看建表语句

ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users MODIFY COLUMN age TINYINT;
ALTER TABLE users DROP COLUMN phone;

DROP TABLE users;                          -- 删除表
TRUNCATE TABLE users;                      -- 清空表（重置自增 ID）
```

### 常用数据类型

```text
数值：
  TINYINT         1 字节   (-128~127 / 0~255)
  INT             4 字节   (-21亿~21亿)
  BIGINT          8 字节
  DECIMAL(10,2)   定点数   (精确小数，用于金额)

字符串：
  CHAR(n)         定长     (性能好，但浪费空间)
  VARCHAR(n)      变长     (常用，最长 65535)
  TEXT            文本     (最大 64KB)
  LONGTEXT        大文本   (最大 4GB)

时间：
  DATE            日期     (2026-07-28)
  DATETIME        日期时间 (2026-07-28 10:00:00)
  TIMESTAMP       时间戳   (带时区，2038 年问题)
```

## 三、CRUD

```sql
-- INSERT
INSERT INTO users (name, email, age) VALUES ('Alice', 'alice@example.com', 25);
INSERT INTO users (name, email, age) VALUES
  ('Bob', 'bob@example.com', 30),
  ('Charlie', 'charlie@example.com', 28);

INSERT INTO users SET name='David', email='david@example.com';

-- SELECT
SELECT * FROM users;
SELECT name, email FROM users WHERE age > 25;
SELECT * FROM users WHERE name LIKE 'A%';
SELECT * FROM users WHERE email IN ('alice@ex.com', 'bob@ex.com');
SELECT * FROM users WHERE age BETWEEN 20 AND 30;
SELECT * FROM users ORDER BY age DESC, created_at ASC LIMIT 10 OFFSET 0;

-- 聚合
SELECT COUNT(*), AVG(age), MAX(age), MIN(age) FROM users;
SELECT status, COUNT(*) FROM users GROUP BY status;
SELECT age, COUNT(*) FROM users GROUP BY age HAVING COUNT(*) > 1;

-- UPDATE
UPDATE users SET age = 26 WHERE name = 'Alice';
UPDATE users SET age = age + 1 WHERE id = 1;

-- DELETE
DELETE FROM users WHERE id = 3;
DELETE FROM users WHERE age < 18;
```

### 查询执行顺序

```text
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

## 四、JOIN

```sql
CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  product VARCHAR(100),
  amount DECIMAL(10, 2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- INNER JOIN：两表都有的数据
SELECT u.name, o.product, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN：左表全保留，右表没有的为 NULL
SELECT u.name, o.product
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 统计每个用户的订单数
SELECT u.name, COUNT(o.id) AS order_count, COALESCE(SUM(o.amount), 0) AS total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- 多表 JOIN
SELECT u.name, o.product, oi.quantity
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id;
```

### JOIN 对比

| JOIN 类型 | 结果 |
|-----------|------|
| INNER JOIN | A ∩ B（交集） |
| LEFT JOIN | A 全保留 + B 匹配的 |
| RIGHT JOIN | B 全保留 + A 匹配的 |
| FULL OUTER JOIN | A ∪ B（MySQL 不支持，用 UNION 模拟） |

## 五、索引

```sql
-- 创建索引
CREATE INDEX idx_email ON users(email);                    -- 单列索引
CREATE UNIQUE INDEX idx_email ON users(email);             -- 唯一索引
CREATE INDEX idx_name_age ON users(name, age);             -- 复合索引
CREATE FULLTEXT INDEX idx_content ON articles(content);    -- 全文索引（MyISAM / InnoDB 5.6+）

-- 查看索引
SHOW INDEX FROM users;

-- 删除索引
DROP INDEX idx_email ON users;
```

### 复合索引的最左前缀原则

```sql
-- 复合索引 (a, b, c)
WHERE a = 1                        -- ✅ 用到索引
WHERE a = 1 AND b = 2              -- ✅ 用到索引
WHERE a = 1 AND b = 2 AND c = 3    -- ✅ 用到索引
WHERE b = 2                        -- ❌ 无法用到索引
WHERE a = 1 AND c = 3              -- 只能用到 a 部分的索引
```

### EXPLAIN 分析查询

```sql
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
```

```text
关键字段：
  type       ALL（全表扫描） / ref（索引等值） / range（范围） / const（主键）
  possible_keys  可能用到的索引
  key            实际用到的索引
  rows           扫描行数
  Extra          Using index（覆盖索引）/ Using filesort（额外排序）
```

## 六、事务

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;     -- 提交
-- ROLLBACK; -- 回滚
```

### ACID 特性

```text
A（Atomicity）原子性：事务要么全部成功，要么全部回滚
C（Consistency）一致性：事务前后数据状态一致
I（Isolation）隔离性：并发事务互不干扰
D（Durability）持久性：提交后数据不会丢失
```

### 隔离级别

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;      -- Oracle 默认
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;     -- MySQL 默认
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|----------|------|-----------|------|
| READ UNCOMMITTED | ✅ | ✅ | ✅ |
| READ COMMITTED | ❌ | ✅ | ✅ |
| REPEATABLE READ | ❌ | ❌ | ✅ |
| SERIALIZABLE | ❌ | ❌ | ❌ |

## 七、SQL 高级查询

```sql
-- 子查询
SELECT * FROM users WHERE id IN (
  SELECT user_id FROM orders WHERE amount > 100
);

-- EXISTS（通常比 IN 快）
SELECT * FROM users u WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.amount > 100
);

-- 窗口函数（MySQL 8.0+）
SELECT
  name,
  age,
  ROW_NUMBER() OVER (ORDER BY age DESC) AS rank,
  RANK() OVER (ORDER BY age DESC) AS rank_with_gap,
  DENSE_RANK() OVER (ORDER BY age DESC) AS dense_rank,
  LAG(age, 1) OVER (ORDER BY age) AS prev_age,
  LEAD(age, 1) OVER (ORDER BY age) AS next_age
FROM users;

-- UNION（合并结果集）
SELECT name, email FROM users
UNION
SELECT name, email FROM deleted_users;

-- CASE WHEN
SELECT
  name,
  CASE
    WHEN age < 18 THEN '未成年'
    WHEN age < 60 THEN '成年'
    ELSE '老年'
  END AS age_group
FROM users;
```

## 八、存储引擎

```sql
SHOW ENGINES;
SHOW TABLE STATUS WHERE Name = 'users';
```

| 特性 | InnoDB（默认） | MyISAM |
|------|---------------|--------|
| 事务 | ✅ | ❌ |
| 外键 | ✅ | ❌ |
| 行级锁 | ✅ | ❌（表级锁） |
| 全文索引 | 5.6+ 支持 | ✅ |
| 崩溃恢复 | ✅ | 需修复 |
| 适用场景 | OLTP 在线事务 | 只读 / 日志表 |

**始终使用 InnoDB**，除非有特殊理由。

## 九、性能优化

```text
SQL 优化原则：
├── 避免 SELECT *，只取需要的列
├── 用 EXPLAIN 检查是否走索引
├── 复合索引注意最左前缀
├── 避免在 WHERE 中对字段做函数操作
│   WHERE YEAR(created_at) = 2026  →  ❌
│   WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'  →  ✅
├── 避免隐式类型转换
│   WHERE phone = 13800138000  →  字符串 vs 数字不匹配
├── 大分页优化
│   LIMIT 100000, 20  →  先查 id 再 JOIN
│   SELECT * FROM users WHERE id > 100000 LIMIT 20
└── 用覆盖索引（Extra: Using index）
```

## 十、面试题

### Q1: MySQL 为什么用 B+ Tree 做索引

```text
1. B+ Tree 非叶子节点只存索引，不存数据，比 B-Tree 更矮宽
   → 磁盘 IO 次数少（3-4 层即可支撑千万级数据）
2. 叶子节点通过链表相连
   → 范围查询（BETWEEN、>、<）不用回树查找，链表遍历即可
3. 所有数据都在叶子节点
   → 查询性能稳定（每次都要到叶子）
```

### Q2: 聚簇索引和非聚簇索引的区别

```text
聚簇索引（Clustered Index）：
  数据和索引存储在一起，InnoDB 的主键就是聚簇索引。
  叶子节点存储整行数据。
  一张表只能有一个聚簇索引。

非聚簇索引（Secondary Index）：
  叶子节点存储主键值。
  通过非聚簇索引查询时，先找到主键，再回表查数据。
  一张表可以有多个非聚簇索引。

覆盖索引：当查询所需字段都在索引中时，无需回表。
```

### Q3: 如何优化慢查询

```text
1. 开启慢查询日志
   SET GLOBAL slow_query_log = ON;
   SET GLOBAL long_query_time = 1;  -- 超过 1 秒

2. 用 EXPLAIN 分析
3. 检查索引：是否无索引 / 索引失效 / 选错索引
4. 优化 SQL：避免函数操作、大分页、SELECT *
5. 读写分离：主库写、从库读
6. 分表分库：单表超过 500 万行考虑分区或分表
```

### Q4: MVCC 是什么

```text
MVCC（Multi-Version Concurrency Control）多版本并发控制。
InnoDB 通过 MVCC 实现 READ COMMITTED 和 REPEATABLE READ 隔离级别。

原理：
  每行数据有隐藏的 trx_id（事务 ID）和 roll_pointer（回滚指针）。
  事务读数据时，根据当前活跃事务列表，判断哪个版本可见。
  不同隔离级别下，判断规则不同。

好处：读不阻塞写，写不阻塞读。
```
