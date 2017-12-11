# Haproxy + MySQL 负载均衡，高可用

[参考连接](http://www.cnblogs.com/suixinpeng/p/5575729.html)

## 机器列表
```
172.18.88.44  tospur-es1 # haproxy keepalived
172.18.88.45  tospur-es2 # mysql haproxy keepalived
172.18.88.46  tospur-es3 # mysql

+-------------+                       +-------------+
|   mysql01   |-----------------------|   mysql02   |
+-------------+                       +-------------+
       |        .                   .         |
       |            .            .            |
       |                .     .               |
       |                   .                  |
       |                .      .              |
       |             .             .          |
       |          .                    .      |
    MASTER     .      keep|alived         BACKUP
172.18.88.44         172.18.88.42      172.18.88.45
+-------------+    +-------------+    +-------------+
| haproxy01   |----|  virtualIP  |----| haproxy02   |
+-------------+    +-------------+    +-------------+
                          |
       +------------------+------------------+
       |                  |                  |
+-------------+    +-------------+    +-------------+
|   client    |    |   client    |    |   client    |
+-------------+    +-------------+    +-------------+
```

## 软件安装
*这里以MySQL为例，MariaDB配置情况相同*
```
172.18.88.44, 172.18.88.45
yum install -y haproxy keepalived

172.18.88.45, 172.18.88.46 
yum install -y mysql-server
```

## Mysql 启动
*两台MySQL机器都需要启动，授权*
```
service mysqld start

授权 haproxy 所在机器(172.18.88.44, 172.18.88.45)能够连接
mysql> GRANT ALL PRIVILEGES ON *.* to 'root'@'172.18.88.44';
mysql> GRANT ALL PRIVILEGES ON *.* to 'root'@'172.18.88.45';
mysql> FLUSH PRIVILEGES;
```

## Haproxy 配置启动
172.18.88.44, 172.18.88.45 同时都需要配置 haproxy(内容一致)  
/etc/haproxy/haproxy.cfg 
```
global
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

############################关键配置##############
listen test1
        bind 0.0.0.0:3307
        mode tcp
        #maxconn 4086
        #log 127.0.0.1 local0 debug
        server s1 172.18.88.45:3306
        server s2 172.18.88.46:3306
############################关键配置##############
```
*注意现在只需要连接 172.18.88.44:3307 或 172.18.88.45:3307 就可以连接msyql*

## 测试 Haproxy
```
在 172.18.88.45 数据库test 下新建表 test45
在 172.18.88.46 数据库test 下新建表 test46

重复连接多次会，会观察到 test 库下的不同表 test45 或 test46
```
```
(连接172.18.88.44, 172.18.88.45 都可以)

mysql -h 172.18.88.44 -P 3307
mysql> use test;
Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test46         |
+----------------+
1 row in set (0.00 sec)


mysql -h 172.18.88.44 -P 3307
mysql> use test;
Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| test45         |
+----------------+
1 row in set (0.00 sec)
```
## Keepalived 配置
### *Master 机器上的配置*
/etc/keepalived/check_haproxy.sh
```bash
#!/bin/bash
counter=$(ps -C haproxy --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    /etc/init.d/haproxy start
    sleep 2
    counter=$(ps -C haproxy --no-heading|wc -l)
    if [ "${counter}" = "0" ]; then
        /etc/init.d/keepalived stop
    fi
fi
```

172.18.88.44
/etc/keepalived/keepalived.conf
```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight -5
    fall 3  
    rise 2
}


vrrp_instance VI_haproxy {
    state MASTER
    interface eth0
    mcast_src_ip 172.18.88.44
    virtual_router_id 50
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.18.88.42
    }
    track_script {
        chk_haproxy
    }
}
```

### Keepalived 备机配置
172.18.88.45  
*/etc/keepalived/check_haproxy.sh* 文件和Master机器相同
```
state MASTER -> state BACKUP，
priority 101 -> priority 100，
mcast_src_ip 172.18.88.44 -> mcast_src_ip 172.18.88.45
```
```
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
}

vrrp_script chk_haproxy {
#    script "killall -0 nginx"
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight -5
    fall 3  
    rise 2
}


vrrp_instance VI_haproxy {
    state BACKUP
    interface eth0
    mcast_src_ip 172.18.88.45
    virtual_router_id 50
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.18.88.42
    }
    track_script {
        chk_haproxy
    }
}
```

## 测试
测试前启动 Keepalived 
```
service keepalived restart
```

```
在Master(172.18.88.44) 上已经可以查看到 浮动ip(172.18.88.42)
ip a | grep eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 172.18.88.44/24 brd 172.18.88.255 scope global eth0
    inet 172.18.88.42/32 scope global eth0

BACKUP机器(172.18.88.44) 
ip a | grep eth0

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 172.18.88.45/24 brd 172.18.88.255 scope global eth0
```
####异常情况测试
在Master 机器上关闭 Keepalived
```
service keepalived stop
```
```
在Master(172.18.88.44) 上浮动Ip 172.18.88.42消失
ip a | grep eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 172.18.88.44/24 brd 172.18.88.255 scope global eth0


BACKUP机器(172.18.88.44) 上浮动Ip 172.18.88.42出现
ip a | grep eth0

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 172.18.88.45/24 brd 172.18.88.255 scope global eth0
    inet 172.18.88.42/32 scope global eth0
```

##最终测试连接浮动 Ip, 建立 mysql 连接
```
成功连接 172.18.88.42
mysql -h 172.18.88.42 -P 3307
```