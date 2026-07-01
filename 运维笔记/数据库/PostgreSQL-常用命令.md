# PostgreSQL 常用命令速查

> 命令行工具：`psql` | 配置文件：`postgresql.conf`、`pg_hba.conf`  
> 官方文档：https://www.postgresql.org/docs/

---

## 一、连接与退出

```bash
# 本地连接
psql -U postgres
psql -U postgres -d mydb
psql -U postgres -h 192.168.1.100 -p 5432 -d mydb

# 指定连接字符串
psql "postgresql://user:password@host:5432/dbname"

# 退出
\q
```

---

## 二、psql 元命令

| 命令 | 说明 |
|------|------|
| `\l` | 列出所有数据库 |
| `\l+` | 列出数据库（含大小、描述） |
| `\c dbname` | 切换数据库 |
| `\conninfo` | 显示当前连接信息 |
| `\dt` | 列出当前库所有表 |
| `\dt+` | 列出表（含大小、描述） |
| `\d tablename` | 查看表结构 |
| `\di` | 列出索引 |
| `\dv` | 列出视图 |
| `\df` | 列出函数 |
| `\du` | 列出用户/角色 |
| `\dn` | 列出 Schema |
| `\dp` | 列出表权限 |
| `\x` | 切换扩展显示（适合宽表） |
| `\timing` | 切换 SQL 执行时间显示 |
| `\e` | 打开编辑器编辑上一条 SQL |
| `\i file.sql` | 执行 SQL 文件 |
| `\o file.txt` | 输出到文件 |
| `\copy` | 导入导出 CSV |
| `\?` | 查看所有 psql 命令 |
| `\h SQL` | 查看 SQL 语法帮助 |

---

## 三、数据库操作

```sql
-- 创建数据库
CREATE DATABASE mydb;
CREATE DATABASE mydb OWNER myuser ENCODING 'UTF8' LC_COLLATE 'zh_CN.UTF-8' LC_CTYPE 'zh_CN.UTF-8';

-- 删除数据库
DROP DATABASE mydb;
DROP DATABASE IF EXISTS mydb;

-- 重命名数据库
ALTER DATABASE oldname RENAME TO newname;

-- 查看数据库大小
SELECT pg_database_size('mydb');
SELECT pg_size_pretty(pg_database_size('mydb'));

-- 查看所有数据库大小
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;
```

---

## 四、用户与权限

```sql
-- 创建用户
CREATE USER myuser WITH PASSWORD 'mypassword';
CREATE ROLE myrole WITH LOGIN PASSWORD 'mypassword';

-- 创建超级用户
CREATE USER admin WITH SUPERUSER PASSWORD 'mypassword';

-- 修改密码
ALTER USER myuser WITH PASSWORD 'newpassword';

-- 删除用户
DROP USER myuser;

-- 授权
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
GRANT ALL ON ALL TABLES IN SCHEMA public TO myuser;
GRANT SELECT, INSERT, UPDATE ON mytable TO myuser;
GRANT USAGE ON SCHEMA public TO myuser;

-- 回收权限
REVOKE ALL ON mytable FROM myuser;

-- 查看用户权限
\du+
SELECT * FROM pg_roles;
```

---

## 五、表操作

```sql
-- 创建表
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE,
    age INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

-- 带约束的建表
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    amount DECIMAL(10,2) CHECK (amount > 0),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- 删除表
DROP TABLE IF EXISTS users CASCADE;

-- 重命名
ALTER TABLE users RENAME TO customers;

-- 添加列
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 删除列
ALTER TABLE users DROP COLUMN phone;

-- 修改列类型
ALTER TABLE users ALTER COLUMN age TYPE BIGINT;

-- 添加约束
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);
ALTER TABLE users ADD CHECK (age >= 0);

-- 查看表大小
SELECT pg_size_pretty(pg_total_relation_size('users'));
SELECT pg_size_pretty(pg_relation_size('users'));
```

---

## 六、索引

```sql
-- 创建索引
CREATE INDEX idx_users_name ON users (name);
CREATE INDEX idx_users_email ON users (email);
CREATE UNIQUE INDEX idx_users_email_unique ON users (email);

-- 复合索引
CREATE INDEX idx_orders_user_status ON orders (user_id, status);

-- 部分索引
CREATE INDEX idx_active_users ON users (name) WHERE status = 'active';

-- 查看索引
\di
SELECT * FROM pg_indexes WHERE tablename = 'users';

-- 删除索引
DROP INDEX idx_users_name;

-- 重建索引
REINDEX TABLE users;
REINDEX DATABASE mydb;
```

---

## 七、查询 (SELECT)

```sql
-- 基础查询
SELECT * FROM users;
SELECT id, name, email FROM users WHERE age > 18;

-- 排序
SELECT * FROM users ORDER BY created_at DESC;

-- 分页
SELECT * FROM users ORDER BY id LIMIT 10 OFFSET 20;

-- 聚合
SELECT status, COUNT(*), SUM(amount), AVG(amount) FROM orders GROUP BY status;

-- 过滤分组
SELECT status, COUNT(*) FROM orders GROUP BY status HAVING COUNT(*) > 5;

-- 连接
SELECT u.name, o.amount FROM users u
INNER JOIN orders o ON u.id = o.user_id;

SELECT u.name, o.amount FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 子查询
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE amount > 100);

-- 去重
SELECT DISTINCT status FROM orders;

-- UNION
SELECT name FROM users UNION SELECT name FROM archived_users;
```

---

## 八、插入、更新、删除

```sql
-- 插入单行
INSERT INTO users (name, email, age) VALUES ('Alice', 'alice@example.com', 25);

-- 插入多行
INSERT INTO users (name, email, age) VALUES
    ('Bob', 'bob@example.com', 30),
    ('Charlie', 'charlie@example.com', 28);

-- 从查询插入
INSERT INTO users_archive SELECT * FROM users WHERE created_at < '2020-01-01';

-- 更新
UPDATE users SET age = 26 WHERE name = 'Alice';
UPDATE users SET age = age + 1, updated_at = NOW() WHERE id = 1;

-- 删除
DELETE FROM users WHERE id = 1;
DELETE FROM users WHERE created_at < '2020-01-01';

-- 清空表
TRUNCATE TABLE users;
TRUNCATE TABLE users CASCADE;   -- 同时清空关联表
```

---

## 九、事务

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- 或
ROLLBACK;

-- 设置保存点
BEGIN;
    INSERT INTO users (name) VALUES ('Dave');
    SAVEPOINT sp1;
    INSERT INTO users (name) VALUES ('Eve');
    ROLLBACK TO sp1;   -- 回滚到保存点，Dave 保留
COMMIT;
```

---

## 十、备份与恢复

```bash
# 备份单个数据库
pg_dump -U postgres -d mydb > mydb.sql
pg_dump -U postgres -d mydb -Fc > mydb.dump    # 压缩格式

# 备份所有数据库
pg_dumpall -U postgres > all.sql

# 只备份表结构
pg_dump -U postgres -d mydb --schema-only > schema.sql

# 只备份数据
pg_dump -U postgres -d mydb --data-only > data.sql

# 恢复
psql -U postgres -d mydb < mydb.sql
pg_restore -U postgres -d mydb mydb.dump
```

---

## 十一、导入导出

```sql
-- 导出 CSV
\copy (SELECT * FROM users) TO '/tmp/users.csv' WITH CSV HEADER;

-- 导入 CSV
\copy users FROM '/tmp/users.csv' WITH CSV HEADER;

-- 导出 JSON
SELECT row_to_json(t) FROM (SELECT * FROM users) t;
SELECT json_agg(row_to_json(t)) FROM (SELECT * FROM users) t;
```

---

## 十二、性能与监控

```sql
-- 查看当前活动连接
SELECT * FROM pg_stat_activity;

-- 杀掉连接
SELECT pg_terminate_backend(pid);

-- 查看锁
SELECT * FROM pg_locks;

-- 查看慢查询日志配置
SHOW log_min_duration_statement;

-- 查看表和索引大小
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC;

-- 查看未使用的索引
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- 查看表统计信息
SELECT * FROM pg_stat_user_tables WHERE relname = 'users';

-- EXPLAIN 分析查询
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@example.com';
```

---

## 十三、Schema 管理

```sql
-- 创建 Schema
CREATE SCHEMA myapp;

-- 切换 Schema
SET search_path TO myapp, public;

-- 删除 Schema
DROP SCHEMA myapp CASCADE;

-- 查看当前 Schema
SHOW search_path;
```

---

## 十四、视图与物化视图

```sql
-- 创建视图
CREATE VIEW active_users AS
SELECT id, name, email FROM users WHERE status = 'active';

-- 创建物化视图
CREATE MATERIALIZED VIEW order_summary AS
SELECT user_id, COUNT(*), SUM(amount)
FROM orders GROUP BY user_id;

-- 刷新物化视图
REFRESH MATERIALIZED VIEW order_summary;

-- 删除视图
DROP VIEW active_users;
```

---

## 十五、序列

```sql
-- 查看序列
SELECT * FROM users_id_seq;

-- 重置序列
ALTER SEQUENCE users_id_seq RESTART WITH 1;

-- 查看当前值
SELECT currval('users_id_seq');
SELECT nextval('users_id_seq');
```

---

## 十六、扩展

```sql
-- 查看已安装扩展
SELECT * FROM pg_extension;

-- 安装扩展
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- 使用 UUID
SELECT uuid_generate_v4();
```

---

## 十七、Docker 中的 PostgreSQL

```bash
# 启动
docker run -d --name pg \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 postgres:16

# 连接
docker exec -it pg psql -U postgres

# 备份
docker exec pg pg_dump -U postgres mydb > backup.sql

# 恢复
docker exec -i pg psql -U postgres mydb < backup.sql
```

---

## 十八、常用函数

| 函数 | 说明 |
|------|------|
| `NOW()` | 当前时间戳 |
| `CURRENT_DATE` | 当前日期 |
| `CURRENT_TIME` | 当前时间 |
| `COALESCE(val, default)` | 空值替换 |
| `NULLIF(a, b)` | 相等返回 NULL |
| `LENGTH(str)` | 字符串长度 |
| `UPPER(str)` / `LOWER(str)` | 大小写转换 |
| `CONCAT(a, b)` | 拼接 |
| `EXTRACT(YEAR FROM date)` | 提取日期部分 |
| `AGE(date1, date2)` | 计算时间间隔 |
| `uuid_generate_v4()` | 生成 UUID |
| `gen_random_uuid()` | 生成随机 UUID (PG 13+) |