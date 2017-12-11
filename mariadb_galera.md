# Mariadb Galera 集群搭建
[参考链接](https://www.unixmen.com/setup-mariadb-galera-cluster-10-0-centos/)

## 机器列表
```
172.18.88.44  tospur-es1
172.18.88.45  tospur-es2
172.18.88.46  tospur-es3
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
启动 mysql 服务
```
service mysql start
```

## MariaDB 安全配置
所有机器  
```
mysql_secure_installation
设置 password 为 'dbpass', 其他选项全部默认(全部选择y)
```

## 创建 MariaDB Galera Cluster 用户
所有机器  

```
mysql -u root -p
输入密码：dbpass
```
```
DELETE FROM mysql.user WHERE user='';
GRANT ALL ON *.* TO 'root'@'%' IDENTIFIED BY 'dbpass';
GRANT USAGE ON *.* to sst_user@'%' IDENTIFIED BY 'dbpass';
GRANT ALL PRIVILEGES on *.* to sst_user@'%';
FLUSH PRIVILEGES;
quit
```

## 修改 MariaDB Galera 集群配置
所有机器
```
# 停止mysql service
service mysql stop
```

在机器 172.18.88.44 上的通过以下命令追加配置文件  
```
cat >> /etc/my.cnf.d/server.cnf << EOF
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
innodb_locks_unsafe_for_binlog=1
query_cache_size=0
query_cache_type=0
bind-address=0.0.0.0
datadir=/var/lib/mysql
innodb_log_file_size=100M
innodb_file_per_table
innodb_flush_log_at_trx_commit=2
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://172.18.88.44,172.18.88.45,172.18.88.46"
wsrep_cluster_name='galera_cluster'
wsrep_node_address='172.18.88.44'
wsrep_node_name='tospur-es1'
wsrep_sst_method=rsync
wsrep_sst_auth=sst_user:dbpass
EOF
```
*这段配置内容会追加在 [mariadb-10.0] 组下面*  
**在其他的机器（172.18.88.45，172.18.88.46）上修改相应的项 wsrep_node_address 和 wsrep_node_name**

172.18.88.45（tospur-es2）
```
wsrep_node_address='172.18.88.45'
wsrep_node_name='tospur-es2'
```

172.18.88.46（tospur-es3）
```
wsrep_node_address='172.18.88.46'
wsrep_node_name='tospur-es3'
```

## 初始化集群首节点
在首节点 172.18.88.44（tospur-es1） 执行以下命令
```
/etc/init.d/mysql start --wsrep-new-cluster
```
查看状态
```
# 需要输入密码 dbpass
mysql -uroot -p -e "show status like 'wsrep%'"
```
如下结果表示成功
```
| wsrep_local_state_comment    | Synced    
| wsrep_incoming_addresses     | 172.18.88.44:3306   
| wsrep_cluster_size           | 1                                             
| wsrep_ready                  | ON                                            
```

## 添加其他节点到集群中
分别在其他节点（172.18.88.45， 172.18.88.46）上启动 mysql   
**记得在启动之前检查/etc/my.cnf.d/server.cnf 是否正确配置**
```
service mysql start
```
检查集群状态
```
# 需要输入密码 dbpass
mysql -uroot -p -e "show status like 'wsrep%'"
```
如下结果表示成功
```
| wsrep_local_state_comment    | Synced    
| wsrep_incoming_addresses     | 172.18.88.46:3306,172.18.88.44:3306,172.18.88.45:3306   
| wsrep_cluster_size           | 3                                             
| wsrep_ready                  | ON    
```

## 验证集群同步数据功能
集群已经成功启动运行，接下来测试功能  
在节点 172.18.88.44（tospur-es1）执行如下命令
```
mysql -u root -p -e 'CREATE DATABASE clustertest;'
```
```
mysql -u root -p -e 'CREATE TABLE clustertest.mycluster ( id INT NOT NULL AUTO_INCREMENT, name VARCHAR(50), ipaddress VARCHAR(20), PRIMARY KEY(id));'
```
```
mysql -u root -p -e 'INSERT INTO clustertest.mycluster (name, ipaddress) VALUES ("db1", "1.1.1.1");'
```
查看数据是否已经插入成功
```
mysql -u root -p -e 'SELECT * FROM clustertest.mycluster;'
```
```
+----+------+-----------+
| id | name | ipaddress |
+----+------+-----------+
|  5 | db1  | 1.1.1.1   |
+----+------+-----------+
```
在节点 172.18.88.45（tospur-es2）上检查数据
```
mysql -u root -p -e 'SELECT * FROM clustertest.mycluster;'
```
```
+----+------+-----------+
| id | name | ipaddress |
+----+------+-----------+
|  5 | db1  | 1.1.1.1   |
+----+------+-----------+
```
在节点 172.18.88.46（tospur-es3）上检查数据
```
mysql -u root -p -e 'SELECT * FROM clustertest.mycluster;'
```
```
+----+------+-----------+
| id | name | ipaddress |
+----+------+-----------+
|  5 | db1  | 1.1.1.1   |
+----+------+-----------+
```

到此，数据库集群已经成功安装