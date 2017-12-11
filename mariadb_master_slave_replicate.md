# Mariadb 主从同步
[参考链接](https://renwole.com/archives/208)
## 机器列表
```
172.18.88.44  tospur-es1 # master
172.18.88.45  tospur-es2 # slave
```

## 关闭防火墙
所有机器  
```
service iptables stop

# Set SELinux in permissive mode
setenforce 0
```

## 添加 MariaDB Repo 文件
所有机器  
/etc/yum.repos.d/MariaDB.repo 
```
[mariadb]
name = MariaDB
baseurl = http://mirrors.ustc.edu.cn/mariadb/yum/10.0/centos6-amd64
gpgkey=http://mirrors.ustc.edu.cn/mariadb/yum/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

## MariaDB Galera 安装
所有机器  
```
yum install -y MariaDB-Galera-server MariaDB-client galera
```

## Master 配置文件配置
172.18.88.44 (tospur-es1)  
/etc/my.cnf.d/server.cnf
```
[mariadb]
binlog-format = row #二进制日志记录的模式
binlog-checksum = CRC32 #可使主机为写入二进制日志的事件写入校验（高版本默认开启）
sync-master-info = 1 #MariaDB依靠操作系统将master.info文件刷新到磁盘。
sync_relay_log_info = 1 #MariaDB依靠操作系统将relay-log.info文件刷新到磁盘。
expire_logs_days = 7 #日志文件过期天数,默认是 0,表示不过期 
master-verify-checksum = 1 #主服务器效验
slave-sql-verify-checksum = 1 #从服务器效验

server-id = 56 #MySQL服务器ID,不重复
log-bin = mysql-bin #二进制日志（默认开启）
sync-binlog = 1 #主服务器进行设置,用于事务安全
```

## Slave 配置文件配置
172.18.88.45 (tospur-es2)  
/etc/my.cnf.d/server.cnf
```
[mariadb]
binlog-format = row #二进制日志记录的模式
binlog-checksum = CRC32 #可使主机为写入二进制日志的事件写入校验（高版本默认开启）
sync-master-info = 1 #MariaDB依靠操作系统将master.info文件刷新到磁盘。
sync_relay_log_info = 1 #MariaDB依靠操作系统将relay-log.info文件刷新到磁盘。
expire_logs_days = 7 #日志文件过期天数,默认是 0,表示不过期 
master-verify-checksum = 1 #主服务器效验
slave-sql-verify-checksum = 1 #从服务器效验

server-id = 163
relay-log = relay-bin #中继日志
slave-parallel-threads = 2 #设定从服务器的SQL线程数
#replicate-do-db = renwoleblogdb#复制指定的数据库,多个写多行
#replicate-ignore-db = mysql #不备份的数据库,多个写多行
relay_log_recovery = 1 #从站崩溃后可以使用,防止损坏的中继日志处理。
log-slave-updates = 1 #slave将复制事件写进自己的二进制日志
```

## 启动 mysql 服务
所有机器
```
service mysql start
```

## MariaDB 安全配置
所有机器  
```
mysql_secure_installation
设置 password 为 'dbpass', 其他选项全部默认(全部选择y)
```

## 主服务器Master授权配置
172.18.88.44 (tospur-es1)
```
# 需要输入密码：dbpass
mysql -uroot -p
```
```
-- 创建Slave专用备份账号
GRANT REPLICATION SLAVE ON *.* TO 'replicater'@'%' IDENTIFIED BY 'dbpass';
flush privileges;

-- 查看授权情况
SELECT DISTINCT CONCAT('User: ''',user,'''@''',host,''';') AS query FROM mysql.user; 

-- 锁定数据库防止master值变化
flush tables with read lock;
-- 获取master状态值
show master status; 
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000002 |      635 |              |                  |
+------------------+----------+--------------+------------------+
```

一旦获取了备份时正确的Binlog位点（文件名和偏移量），那么就可以用BINLOG_GTID_POS()函数来计算GTID

```
SELECT BINLOG_GTID_POS("mysql-bin.000002", 635);
+------------------------------------------+
| BINLOG_GTID_POS("mysql-bin.000002", 635) |
+------------------------------------------+
| 0-56-2                                   |
+------------------------------------------+
```

## 从服务器Slave 配置
172.18.88.45 (tospur-es2)   
*正如官方所说从MariaDB 10.0.13版本开始，新的SLAVE可以通过设置 @@gtid_slave_pos 的值来设定复制的起始位置，用 CHANGE MASTER 把这个值传给主库，然后开始复制*
```
# 需要输入密码：dbpass
mysql -uroot -p
```
```
-- 记得修改为你当前系统的值
SET GLOBAL gtid_slave_pos = "0-56-2"; 
-- 进行主从授权(记得修改相应的host ip)
change master to master_host='172.18.88.44',MASTER_PORT = 3306,master_user='replicater',master_password='dbpass',master_use_gtid=slave_pos; 
START SLAVE;
show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.18.88.44
                  Master_User: replicater
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 635
               Relay_Log_File: relay-bin.000002
                Relay_Log_Pos: 654
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
            ..........................
                   Using_Gtid: Slave_Pos
                  Gtid_IO_Pos: 0-56-2

```
如果 Slave_IO_Running 与 Slave_SQL_Running 都为YES,表明从服务已经运行，Using_Gtid列判断GTID值是否一致。
```
说明：
master_host 表示master授权地址
MASTER_PORT MySQL端口
master_user 表示master授权账号
master_password 表示密码
master_use_gtid GTID变量值
```

## 接下来解锁主服务器数据库表
```
# 需要输入密码：dbpass
mysql -uroot -p
```
```
-- 解锁数据表
unlock tables; 
-- 查看从服务器连接状态
show slave hosts; 
-- 查看客户端
show global status like "rpl%"; 
```

## 从服务器Slave查看relay的所有相关参数
```
show variables like '%relay%';
```

#### 主从已经配置完成。现在无论在主服务器上增、改、删、查，都会同步到从服务器，根据自己的需求进行相关测试即可。