---
layout: post
title: "mysql主从同步"
category: CI/DI
---

# mysql主从同步

## 需求

有一个主库和一个从库，从库需要同步主库的某张表

## 实现

采用mysql的binlong进行同步

**局限性**：如果只同步一张表，那么库名表名必须一样，否者如果库名不同，那么只能整个库全量同步所有表

## 技术选择

### 传统复制的流程

**主库写 binlog**
 主库执行更新类 SQL（INSERT、UPDATE、DELETE 等）时，把这些语句（或行事件）记录到 binlog（binary log，二进制日志文件）。

**从库 IO 线程**
 从库启动复制后，会创建一个 IO 线程，连接到主库，请求从某个 `binlog 文件名` + `位置点 (log_pos)` 开始的日志内容。

**主库 dump 线程**
 主库为这个从库开启一个 dump 线程，把对应位置之后的 binlog 内容发送给从库。

**从库 relay log**
 从库 IO 线程把收到的 binlog 数据写入中继日志（relay log）。

**从库 SQL 线程**
 从库的 SQL 线程读取 relay log，将其“重放”，执行成真正的 SQL，从而让数据与主库保持一致

依赖 binlog 文件名 + 文件位置

```
CHANGE MASTER TO
  MASTER_HOST='192.168.1.10',
  MASTER_USER='repl',
  MASTER_PASSWORD='123456',
  MASTER_LOG_FILE='mysql-bin.000005',
  MASTER_LOG_POS=120;
```

需要人工指定 binlog 文件名和位置，容易出错。

如果出现 binlog 日志缺失或文件切换，容易导致复制中断。

### 基于 **GTID (Global Transaction Identifier)** 

**故障切换更方便（简化运维）**

主库会自动找到正确的事务位置，避免人为出错。

**便于监控和管理**

- 可以通过 `SHOW SLAVE STATUS\G` 直接看到 `Retrieved_Gtid_Set` 和 `Executed_Gtid_Set`，方便定位复制延迟、同步状态。

**总结：** 选择GTID



## **实践**

在docker中运行2两个 mysql实例 一主一从

```shell
#网络互通
docker network create mysql-net

# 主库
docker run -d \
  --name mysql-master \
  --network mysql-net \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -p 3307:3306 \
  mysql:8.0.22
  
  # 从库
docker run -d \
  --name mysql-slave \
  --network mysql-net \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -p 3308:3306 \
  mysql:8.0.22
```

### 主库配置

配置主库 my.cnf  开启gtid同步，然后重启 mysql 服务

```
server-id=1
log-bin=mysql-bin
binlog_format=ROW
gtid_mode=ON
enforce_gtid_consistency=ON
skip-slave-start=ON
```

1. 给已有的用户授权

```mysql
mysql -u root -p
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root123';
GRANT REPLICATION SLAVE ON *.* TO 'root'@'%';
FLUSH PRIVILEGES;
```

2. 或者创建一个支持同步的用户

```mysql
mysql -u root -p
CREATE USER 'repl'@'%' IDENTIFIED BY 'your_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

导出数据,binlog是增量同步的 因此需要先在服务器中使用mysqldump命令来导出初始表的数据

```

mysqldump -uroot -p test1 t1  > /home/mysql/master-t1-table.sql
```



### 从库配置

配置从库 my.cnf  开启gtid同步，然后重启 mysql 服务

```
server-id=2
gtid_mode=ON
enforce_gtid_consistency=ON
skip-slave-start=ON
replicate-do-table=test1.t1  #库名.表名
```

导入数据，初始化原始数据

```
mysql -uroot -p test1< /home/mysql/master-t1-table.sql
```

开始从库同步

```mysql
mysql -u root -p

STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST='mysql-master',
  MASTER_USER='root',
  MASTER_PASSWORD='root123',
  MASTER_AUTO_POSITION = 1;


START SLAVE;
SHOW SLAVE STATUS\G;

```

SHOW SLAVE STATUS 查看同步情况

| 字段                      | 当前值                                                       | 含义                                           |
| ------------------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| `Slave_IO_Running`        | Yes                                                          | 从库 I/O 线程正常连接主库                      |
| `Slave_SQL_Running`       | Yes                                                          | SQL 线程正常执行 relay log                     |
| `Last_IO_Error`           | 空                                                           | 没有 I/O 错误                                  |
| `Last_SQL_Error`          | 空                                                           | 没有 SQL 执行错误                              |
| `Seconds_Behind_Master`   | 0                                                            | 从库已追上主库，没有延迟                       |
| `Slave_SQL_Running_State` | Slave has read all relay log; waiting for more updates       | SQL 线程已执行完所有 relay log，正在等待新事务 |
| `Replicate_Do_Table`      | test1.t1                                                     | 只同步了这个表                                 |
| `Retrieved_Gtid_Set`      | e78e92cf-7fd9-11f0-b13c-663bea67ec9f:5                       | 已从主库获取的 GTID 事务                       |
| `Executed_Gtid_Set`       | e45e4d9f-7fd9-11f0-b26a-0659a8637d18:1-2, e78e92cf-7fd9-11f0-b13c-663bea67ec9f:1-5 | 从库已执行的 GTID 事务                         |

