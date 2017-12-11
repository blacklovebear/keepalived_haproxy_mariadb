# Keepalived + Nginx + Tomcat 双机热备，负载均衡

[参考连接1](http://seanlook.com/2015/05/18/nginx-keepalived-ha/)
[参考连接2](https://segmentfault.com/a/1190000007803704)

## 机器列表
```
172.18.88.44  tospur-es1
172.18.88.45  tospur-es2

+-------------+                       +-------------+
|  tomcat01   |-----------------------|  tomcat02   |
+-------------+                       +-------------+
       |        .                   .         |
       |            .            .            |
       |                .     .               |
       |                   .                  |
       |                .      .              |
       |             .             .          |
       |          .                    .      |
    MASTER            keep|alived         BACKUP
172.18.88.44         172.18.88.43      172.18.88.45
+-------------+    +-------------+    +-------------+
|   nginx01   |----|  virtualIP  |----|   nginx02   |
+-------------+    +-------------+    +-------------+
                          |
       +------------------+------------------+
       |                  |                  |
+-------------+    +-------------+    +-------------+
|    web01    |    |    web02    |    |    web03    |
+-------------+    +-------------+    +-------------+
```

## 软件安装

```
172.18.88.44, 172.18.88.45
yum install -y keepalived nginx
```


## Keepalived 配置
### *Master 机器上的配置*
/etc/keepalived/check_nginx.sh
```bash
#!/bin/bash
counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    nginx
    sleep 2
    counter=$(ps -C nginx --no-heading|wc -l)
    if [ "${counter}" = "0" ]; then
        /etc/init.d/keepalived stop
    fi
fi
```

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

vrrp_script chk_nginx {
#    script "killall -0 nginx"
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -5
    fall 3  
    rise 2
}


vrrp_instance VI_Nginx {
    state MASTER
    interface eth0
    mcast_src_ip 172.18.88.44
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.18.88.43
    }
    track_script {
        chk_nginx
    }
}
```

### Keepalived 备机配置
172.18.88.45  
*/etc/keepalived/check_nginx.sh* 文件和Master机器相同
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

vrrp_script chk_nginx {
#    script "killall -0 nginx"
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -5
    fall 3  
    rise 2
}


vrrp_instance VI_Nginx {
    state BACKUP
    interface eth0
    mcast_src_ip 172.18.88.45
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        172.18.88.43
    }
    track_script {
        chk_nginx
    }
}
```

## Keepalived 测试
测试前启动 Keepalived 
```
service keepalived restart
```

```
在Master(172.18.88.44) 上已经可以查看到 浮动ip(172.18.88.43)
ip a | grep eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 172.18.88.44/24 brd 172.18.88.255 scope global eth0
    inet 172.18.88.43/32 scope global eth0

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
在Master(172.18.88.44) 上浮动Ip 172.18.88.43消失
ip a | grep eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 172.18.88.44/24 brd 172.18.88.255 scope global eth0


BACKUP机器(172.18.88.44) 上浮动Ip 172.18.88.43出现
ip a | grep eth0

2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    inet 172.18.88.45/24 brd 172.18.88.255 scope global eth0
    inet 172.18.88.43/32 scope global eth0
```

**如果再在master机器上启动keepalived 浮动ip又可以回到master机器上**

## Nginx 访问
异常情况下（Keepalived 挂掉）Nginx 通过浮动ip也能正常访问
```
curl http://172.18.88.43
```

## Tomcat 安装启动
172.18.88.44, 172.18.88.45
```
wget http://apache.fayea.com/tomcat/tomcat-8/v8.5.9/bin/apache-tomcat-8.5.9.tar.gz
tar zxvf apache-tomcat-8.5.9.tar.gz 

sudo mv apache-tomcat-8.5.9/ /usr/local/tomcat
关于Tomcat的配置以及设置普通用户等在这里就不提了。直接启动Tomcat。

sudo /usr/local/tomcat/bin/startup.sh
```

#### 测试访问，看Tomcat是否启动成功
```
curl http://172.18.88.44:8080
curl http://172.18.88.45:8080
```

## Nginx + Tomcat 反向代理，负载均衡配置
172.18.88.44, 172.18.88.45 两台机器的Nginx 相同配置  
/etc/nginx/conf.d/default.conf
```
upstream backend_tomcat {
        server 172.18.88.44:8080;
        server 172.18.88.45:8080;
}

server {
        listen 8088; # 根据实际情况选择合适端口
        location / {
                proxy_pass http://backend_tomcat;
        }
}
```

#### Nginx 测试，看是否能成功访问
```
curl http://172.18.88.44:8088
curl http://172.18.88.45:8088
```

## 最终测试
访问浮动IP 172.18.88.43:8088 看是否能成功访问到 Tomcat 服务
```
curl http://172.18.88.43:8088
```

