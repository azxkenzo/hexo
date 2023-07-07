---
title: SQLite
date: 2023-04-01 12:32:50
tags:
---

## 什么是 SQLite？
SQLite是一个进程内的库，实现了自给自足的、无服务器的、零配置的、事务性的 SQL 数据库引擎。它是一个零配置的数据库，这意味着与其他数据库不一样，您不需要在系统中配置。

就像其他数据库，SQLite 引擎不是一个独立的进程，可以按应用程序需求进行静态或动态连接。SQLite 直接访问其存储文件。

## SQLite 语法

### 大小写敏感性
SQLite 是不区分大小写的，但也有一些命令是大小写敏感的，比如 GLOB 和 glob 在 SQLite 的语句中有不同的含义。

### SQLite Insert 语句
INSERT INTO 语句用于向数据库的某个表中添加新的数据行。

INSERT INTO 语句有两种基本语法，如下所示：
```
INSERT INTO TABLE_NAME [(column1, column2, column3,...columnN)]  
VALUES (value1, value2, value3,...valueN);
```

在这里，column1, column2,...columnN 是要插入数据的表中的列的名称。

如果要为表中的所有列添加值，您也可以不需要在 SQLite 查询中指定列名称。但要确保值的顺序与列在表中的顺序一致。SQLite 的 INSERT INTO 语法如下：
```
INSERT INTO TABLE_NAME VALUES (value1,value2,value3,...valueN);
```






