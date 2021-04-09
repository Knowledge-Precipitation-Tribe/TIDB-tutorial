# SQL基本操作

成功部署 TiDB 集群之后，便可以在 TiDB 中执行 SQL 语句了。因为 TiDB 兼容 MySQL，你可以使用 MySQL 客户端连接 TiDB，并且[大多数情况下](https://docs.pingcap.com/zh/tidb/stable/mysql-compatibility)可以直接执行 MySQL 语句。

SQL 是一门声明性语言，它是数据库用户与数据库交互的方式。它更像是一种自然语言，好像在用英语与数据库进行对话。本文档介绍基本的 SQL 操作。完整的 SQL 语句列表，参见 [TiDB SQL 语法详解](https://pingcap.github.io/sqlgram/)。

# 数据库操作

## 查看使用数据库

```sql
SHOW DATABASES;
USE database_name;
```

## 创建数据库

```sql
CREATE DATABASE db_name [operations];
CREATE DATABASE IF NOT EXISTS samp_db; #使用IF NOT EXIST防止发生错误
```

## 删除数据库

```sql
DROP DATABASE samp_db;
```

## 显示数据库中数据表

```sql
SHOW TABLES FROM mysql;
```

# 数据表操作

## 创建数据表

```sql
CREATE TABLE person (
    id INT(11),
    name VARCHAR(255),
    birthday DATE
    );
```

## 显示创建数据表DDL

```sql
SHOW CREATE TABLE person;
```

## 删除数据表

```sql
DROP TABLE person;
```

# 索引操作

## 创建索引

```sql
CREATE INDEX person_id ON person (id);
ALTER TABLE person ADD INDEX person_id (id);
```

对于值唯一的列，可以创建唯一索引。例如：

```sql
CREATE UNIQUE INDEX person_unique_id ON person (id);
ALTER TABLE person ADD UNIQUE person_unique_id (id);
```

## 查看索引

```sql
SHOW INDEX FROM person;
```

## 删除索引

```sql
DROP INDEX person_id ON person;
ALTER TABLE person DROP INDEX person_unique_id;
```

# 记录的增删改查

## 插入记录

```sql
INSERT INTO person VALUES('1','tom','20170912');
INSERT INTO person(id,name) VALUES('2','bob');
```

## 更新记录

```sql
UPDATE person SET birthday='20180808' WHERE id=2;
```

## 删除记录

```sql
DELETE FROM person WHERE id=2;
```

## 查询记录

```sql
SELECT * FROM person;
SELECT name FROM person;
SELECT * FROM person WHERE id<5;
```

# 用户操作

## 创建用户

```sql
CREATE USER 'tiuser'@'localhost' IDENTIFIED BY '123456';
```

## 授权用户

```sql
GRANT SELECT ON samp_db.* TO 'tiuser'@'localhost';
```

## 查看用户权限

```sql
SHOW GRANTS for tiuser@localhost;
```

## 删除用户

```sql
DROP USER 'tiuser'@'localhost';
```

