# MySQL 常用命令速查

> 命令行工具：`mysql` | 配置文件：`/etc/my.cnf`、`~/.my.cnf`  
> 官方文档：https://dev.mysql.com/doc/

---

## 一、连接与退出

```bash
# 本地连接
mysql -u root -p
mysql -u root -p -D mydb

# 远程连接
mysql -u root -p -h 192.168.1.100 -P 3306 -D mydb

# 不提示输入密码（不安全，仅脚本用）
mysql -u root -p'password' -D mydb

# 退出
exit
\q
```

---

## 二、数据库操作

```sql
-- 查看所有数据库
SHOW DATABASES;

-- 创建数据库
CREATE DATABASE mydb;
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 删除数据库
DROP DATABASE IF EXISTS mydb;

-- 切换数据库
USE mydb;

-- 查看当前数据库
SELECT DATABASE();

-- 查看数据库大小
SELECT table_schema AS `database`,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS `size_mb`
FROM information_schema.tables
GROUP BY table_schema;
```

---

## 三、用户与权限

```sql
-- 创建用户
CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'mypassword';
CREATE USER 'myuser'@'%' IDENTIFIED BY 'mypassword';          -- 允许远程

-- 修改密码
ALTER USER 'myuser'@'localhost' IDENTIFIED BY 'newpassword';
SET PASSWORD FOR 'myuser'@'localhost' = 'newpassword';        -- 旧版

-- 授权
GRANT ALL PRIVILEGES ON mydb.* TO 'myuser'@'localhost';
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'myuser'@'localhost';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;

-- 查看权限
SHOW GRANTS FOR 'myuser'@'localhost';

-- 回收权限
REVOKE SELECT ON mydb.* FROM 'myuser'@'localhost';

-- 删除用户
DROP USER 'myuser'@'localhost';

-- 刷新权限
FLUSH PRIVILEGES;

-- 查看所有用户
SELECT user, host FROM mysql.user;
```

---

## 四、表操作

```sql
-- 创建表
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    age TINYINT DEFAULT 0,
    status ENUM('active', 'inactive', 'banned') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 查看所有表
SHOW TABLES;

-- 查看表结构
DESC users;
DESCRIBE users;
SHOW COLUMNS FROM users;
SHOW CREATE TABLE users;

-- 重命名表
RENAME TABLE users TO customers;
ALTER TABLE users RENAME TO customers;

-- 添加列
ALTER TABLE users ADD COLUMN phone VARCHAR(20) AFTER email;

-- 删除列
ALTER TABLE users DROP COLUMN phone;

-- 修改列
ALTER TABLE users MODIFY COLUMN age INT;
ALTER TABLE users CHANGE COLUMN age user_age INT;

-- 添加索引
ALTER TABLE users ADD INDEX idx_name (name);
ALTER TABLE users ADD UNIQUE INDEX idx_email (email);

-- 删除表
DROP TABLE IF EXISTS users;

-- 查看表大小
SELECT table_name,
       ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema = 'mydb' AND table_name = 'users';
```

---

## 五、查询 (SELECT)

```sql
-- 基础
SELECT * FROM users;
SELECT id, name, email FROM users WHERE age > 18;

-- 排序
SELECT * FROM users ORDER BY created_at DESC;
SELECT * FROM users ORDER BY age DESC, name ASC;

-- 分页
SELECT * FROM users LIMIT 10 OFFSET 20;
SELECT * FROM users LIMIT 20, 10;    -- 偏移量, 行数

-- 聚合
SELECT status, COUNT(*), AVG(age), MAX(age), MIN(age) FROM users GROUP BY status;

-- 过滤分组
SELECT status, COUNT(*) FROM users
GROUP BY status HAVING COUNT(*) > 5;

-- 连接
SELECT u.name, o.amount FROM users u
INNER JOIN orders o ON u.id = o.user_id;

SELECT u.name, o.amount FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 子查询
SELECT * FROM users WHERE id IN (
    SELECT user_id FROM orders WHERE amount > 100
);

-- 去重
SELECT DISTINCT status FROM users;

-- UNION
SELECT name FROM users UNION SELECT name FROM archived_users;
SELECT name FROM users UNION ALL SELECT name FROM archived_users;  -- 不去重

-- 模糊查询
SELECT * FROM users WHERE name LIKE 'A%';
SELECT * FROM users WHERE name LIKE '%son';

-- 正则
SELECT * FROM users WHERE name REGEXP '^[A-C]';

-- 范围
SELECT * FROM users WHERE age BETWEEN 18 AND 30;
SELECT * FROM users WHERE id IN (1, 2, 3, 5);
```

---

## 六、插入、更新、删除

```sql
-- 插入
INSERT INTO users (name, email, age) VALUES ('Alice', 'alice@example.com', 25);

-- 插入多行
INSERT INTO users (name, email, age) VALUES
    ('Bob', 'bob@example.com', 30),
    ('Charlie', 'charlie@example.com', 28);

-- 插入或更新（ON DUPLICATE KEY）
INSERT INTO users (id, name, email) VALUES (1, 'Alice', 'new@example.com')
ON DUPLICATE KEY UPDATE name = VALUES(name), email = VALUES(email);

-- 替换（有则删旧行再插入）
REPLACE INTO users (id, name, email) VALUES (1, 'Alice', 'new@example.com');

-- 从查询插入
INSERT INTO users_archive SELECT * FROM users WHERE created_at < '2020-01-01';

-- 更新
UPDATE users SET age = 26 WHERE name = 'Alice';
UPDATE users SET age = age + 1 WHERE id IN (1, 2, 3);

-- 删除
DELETE FROM users WHERE id = 1;
DELETE FROM users WHERE created_at < '2020-01-01';

-- 清空表
TRUNCATE TABLE users;          -- 重置自增 ID，更快
DELETE FROM users;              -- 不重置自增 ID，可回滚
```

---

## 七、事务

```sql
-- InnoDB 支持事务
START TRANSACTION;
-- 或
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
-- 或
ROLLBACK;

-- 设置保存点
SAVEPOINT sp1;
ROLLBACK TO SAVEPOINT sp1;

-- 查看自动提交
SHOW VARIABLES LIKE 'autocommit';
SET autocommit = 0;    -- 关闭自动提交
```

---

## 八、索引

```sql
-- 创建索引
CREATE INDEX idx_users_name ON users (name);
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- 复合索引
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- 全文索引
ALTER TABLE articles ADD FULLTEXT INDEX idx_title_body (title, body);

-- 查看索引
SHOW INDEX FROM users;
SHOW KEYS FROM users;

-- 删除索引
DROP INDEX idx_users_name ON users;
ALTER TABLE users DROP INDEX idx_users_name;

-- 分析索引使用
EXPLAIN SELECT * FROM users WHERE name = 'Alice';
```

---

## 九、备份与恢复

```bash
# 备份单个数据库
mysqldump -u root -p mydb > mydb.sql
mysqldump -u root -p mydb --routines --triggers --events > mydb_full.sql

# 备份所有数据库
mysqldump -u root -p --all-databases > all.sql

# 只备份表结构
mysqldump -u root -p mydb --no-data > schema.sql

# 只备份数据
mysqldump -u root -p mydb --no-create-info > data.sql

# 备份指定表
mysqldump -u root -p mydb users orders > tables.sql

# 恢复
mysql -u root -p mydb < mydb.sql
mysql -u root -p < mydb.sql    # 当 SQL 文件包含 CREATE DATABASE

# 压缩备份与恢复
mysqldump -u root -p mydb | gzip > mydb.sql.gz
gunzip < mydb.sql.gz | mysql -u root -p mydb
```

---

## 十、导入导出

```sql
-- 导出 CSV
SELECT * FROM users
INTO OUTFILE '/tmp/users.csv'
FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n';

-- 导入 CSV
LOAD DATA INFILE '/tmp/users.csv'
INTO TABLE users
FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n'
IGNORE 1 ROWS;    -- 跳过标题行
```

---

## 十一、性能与监控

```sql
-- 查看当前连接
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- 杀掉连接
KILL 123;    -- 进程 ID

-- 查看状态
SHOW STATUS;
SHOW STATUS LIKE 'Threads%';
SHOW STATUS LIKE '%connect%';

-- 查看变量
SHOW VARIABLES;
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE '%buffer%';

-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 查看表大小
SELECT table_schema, table_name,
       ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
ORDER BY data_length + index_length DESC;

-- 查看锁
SHOW OPEN TABLES WHERE In_use > 0;

-- 查看 InnoDB 状态
SHOW ENGINE INNODB STATUS\G
```

---

## 十二、复制

```sql
-- 查看主库状态
SHOW MASTER STATUS;

-- 查看从库状态
SHOW SLAVE STATUS\G

-- 启动/停止复制
START SLAVE;
STOP SLAVE;

-- 重置复制
RESET SLAVE;
RESET MASTER;
```

---

## 十三、日志

```sql
-- 查看 binlog 列表
SHOW BINARY LOGS;
SHOW MASTER LOGS;

-- 查看 binlog 事件
SHOW BINLOG EVENTS IN 'binlog.000001';

-- 查看 binlog 内容（命令行）
mysqlbinlog binlog.000001

-- 按时间点恢复
mysqlbinlog --start-datetime="2024-01-01 10:00:00" \
            --stop-datetime="2024-01-01 11:00:00" \
            binlog.000001 | mysql -u root -p
```

---

## 十四、常用函数

### 字符串

| 函数 | 说明 |
|------|------|
| `CONCAT(a, b)` | 拼接 |
| `CONCAT_WS(',', a, b)` | 用分隔符拼接 |
| `LENGTH(str)` | 字节长度 |
| `CHAR_LENGTH(str)` | 字符长度 |
| `UPPER(str)` / `LOWER(str)` | 大小写 |
| `TRIM(str)` | 去空格 |
| `SUBSTRING(str, 1, 5)` | 截取 |
| `REPLACE(str, 'a', 'b')` | 替换 |
| `INSTR(str, 'abc')` | 查找位置 |
| `LPAD(str, 10, '0')` | 左填充 |

### 日期时间

| 函数 | 说明 |
|------|------|
| `NOW()` | 当前日期时间 |
| `CURDATE()` | 当前日期 |
| `CURTIME()` | 当前时间 |
| `DATE_ADD(NOW(), INTERVAL 7 DAY)` | 日期加 |
| `DATE_SUB(NOW(), INTERVAL 1 MONTH)` | 日期减 |
| `DATEDIFF('2024-01-01', '2023-12-25')` | 日期差 |
| `DATE_FORMAT(NOW(), '%Y-%m-%d')` | 格式化 |
| `UNIX_TIMESTAMP()` | 转 Unix 时间戳 |
| `FROM_UNIXTIME(1700000000)` | Unix 时间戳转日期 |

### 数学

| 函数 | 说明 |
|------|------|
| `ABS(x)` | 绝对值 |
| `ROUND(x, 2)` | 四舍五入 |
| `CEIL(x)` / `FLOOR(x)` | 向上/向下取整 |
| `RAND()` | 随机数 |
| `MOD(10, 3)` | 取模 |

### 条件

| 函数 | 说明 |
|------|------|
| `IF(condition, a, b)` | 条件判断 |
| `IFNULL(val, default)` | 空值替换 |
| `COALESCE(a, b, c)` | 返回第一个非空值 |
| `CASE WHEN ... THEN ... END` | 多条件判断 |

---

## 十五、Docker 中的 MySQL

```bash
# 启动
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=*** \
  -e MYSQL_DATABASE=mydb \
  -e MYSQL_USER=myuser \
  -e MYSQL_PASSWORD=*** \
  -p 3306:3306 \
  mysql:8.0

# 连接
docker exec -it mysql mysql -u root -p

# 备份
docker exec mysql mysqldump -u root -p mydb > backup.sql

# 恢复
docker exec -i mysql mysql -u root -p mydb < backup.sql

# 查看日志
docker logs mysql
```