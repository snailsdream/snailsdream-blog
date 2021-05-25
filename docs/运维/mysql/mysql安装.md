---
title: mysql安装
date: 2021-05-25 22:47:09
permalink: /pages/4c9036/
categories:
  - 运维
  - mysql
tags:
  - mysql
---

记录一次mysql的主从安装过程，本次安装版本为：5.7.26，以rpm的形式进行安装

<!--more-->
## 说明

本次安装以root用户进行，建议为mysql专门建个用户进行安装运行
机器ip信息 ：

1. `*.*.*.14`
2. `*.*.*.15`
  
## 开放端口

```shell
firewall-cmd --add-port=3306/tcp --permanent &&  firewall-cmd --reload
```

## 升级系统

```shell
yum update -y
```

## 安装

```shell
mkdir /data/mysql-rpm && tar -xvf /data/mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar -C /data/mysql-rpm #解压
yum install -y openssl-devel net-tools perl-JSON autoconf #安装依赖
rpm -qa|grep mariadb #查询是否安装了mariadb，如果安装了则需要卸载
rpm -e --nodeps mariadb-libs-5.5.68-1.el7.x86_64 #卸载
cd /data/mysql-rpm && rpm -ivh mysql-community-{common,libs,libs-compat,client,devel,server,embedded-compat,embedded-devel,embedded}-5.7.26-1.el7.x86_64.rpm #安装

```

## 创建数据目录

```shell
mkdir -p /data/mysql/data
```



## 修改配置文件

```shell
vim /etc/my.cnf
```

1. 20行：`datadir=/data/mysql/data`
2. 26行：`log-error=/data/mysql/mysqld.log`
3. 28行：`lower_case_table_names=1`

## 初始化数据库

```shell
mysqld --initialize --user=root --datadir=/data/mysql/data --explicit_defaults_for_timestamp  && chmod -R 777 /data/mysql/*
```

## 启动

```shell
systemctl start  mysqld
```

## 查看默认密码

```shell
grep 'temporary password' /data/mysql/mysqld.log

```

## 修改默认密码

使用默认密码登录后修改密码

```shell
mysql -uroot -p
ALTER USER user() IDENTIFIED BY '*******';

```

## 创建应用数据库

```mysql
create user test@'%' IDENTIFIED WITH 'mysql_native_password' BY '******';
create database test CHARACTER SET utf8 COLLATE utf8_general_ci;
grant all privileges on test.* to test;
```

## 配置主从复制

以下操作需要连接数据库后操作

1. 创建用于同步数据的账号

```shell
grant replication slave on *.* to 'replicate' identified by '********';
flush privileges;
```

### 14数据库

#### 配置修改

vim /etc/my.cnf

```shell
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html
[client]
default_character_set=utf8mb4
[mysqld]
collation_server = utf8mb4_general_ci
character_set_server = utf8mb4

#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
log_bin=mysql-bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/data/mysql/data
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/data/mysql/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid


binlog-format=mixed

#主标服务标识号,必需唯一
server-id=14
# 忽略大小写
lower_case_table_names=1
# 记录日志的数据库
binlog-do-db=test

#不记录日志的库,即不需要同步的库

binlog-ignore-db=mysql
#用从属服务器上的日志功能

log-slave-updates

#经过1日志写操作就把日志文件写入硬盘一次(对日志信息进行一次同步)。n=1是最安全的做法，但效率最低。默认设置是n=0。

sync_binlog=1

# auto_increment，控制自增列AUTO_INCREMENT的行为

# auto_increment_offset＝1设置步长,这里设置为1,这样Master的auto_increment字段产生的数值是:1, 3, 5, 7, …等奇数ID
auto_increment_offset=1
# auto_increment_increment=n有多少台服务器，n就设置为多少
auto_increment_increment=2

#进行镜像处理的数据库

replicate-do-db = test

#不进行镜像处理的数据库

replicate-ignore-db= mysql
```



#### 查看15数据库偏移

root账号进入15数据库查看偏移量

```shell
mysql > show master status;
```

```mysql
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      154 | test         | mysql            |                   |
+------------------+----------+--------------+------------------+-------------------+

```



#### 14数据库设置同步

其中master_log_file和master_log_pos的值来源于另外一台数据库的偏移量结果

```mysql
stop slave;
change master to master_host='*.*.*.15', master_user='replicate', master_password='*******', master_log_file='mysql-bin.000003', master_log_pos=154;
start slave;
show slave status\G;
```

### 15数据库

#### 配置修改

vim /etc/my.cnf

```shell
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[client]
default_character_set=utf8mb4
[mysqld]
collation_server = utf8mb4_general_ci
character_set_server = utf8mb4

#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
log_bin=mysql-bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/data/mysql/data
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/data/mysql/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid


binlog-format=mixed

#主标服务标识号,必需唯一
server-id=15
# 忽略大小写
lower_case_table_names=1
# 记录日志的数据库
binlog-do-db=test

#不记录日志的库,即不需要同步的库

binlog-ignore-db=mysql
#用从属服务器上的日志功能

log-slave-updates

#经过1日志写操作就把日志文件写入硬盘一次(对日志信息进行一次同步)。n=1是最安全的做法，但效率最低。默认设置是n=0。

sync_binlog=1

# auto_increment，控制自增列AUTO_INCREMENT的行为
# auto_increment_offset＝1设置步长,这里设置为1,这样Master的auto_increment字段产生的数值是:1, 3, 5, 7, …等奇数ID
auto_increment_offset=1
# auto_increment_increment=n有多少台服务器，n就设置为多少
auto_increment_increment=2

#进行镜像处理的数据库

replicate-do-db = test

#不进行镜像处理的数据库

replicate-ignore-db= mysql
```



#### 查看14数据库偏移

root账号进入14数据库查看偏移量

```shell
mysql > show master status;
```

```mysql
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |  1258644 | test         | mysql            |                   |
+------------------+----------+--------------+------------------+-------------------
```



#### 15数据库设置同步

```mysql
stop slave;
change master to master_host='*.*.*.14', master_user='replicate', master_password='********', master_log_file='mysql-bin.000004', master_log_pos=1258644;
start slave;
show slave status\G;
```