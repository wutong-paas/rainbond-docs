---
title: Keepalived 部署
description: 在 CentOS 或 Ubuntu 上部署 Keepalived 服务
keywords:
- 在 CentOS 或 Ubuntu 上部署 Keepalived 服务
---

## 前提

:::warning
局域网内交换机需要支持 VRRP 协议，否则 Keepalived 无法正常工作。
:::

在所有网关节点安装 Keepalived

```bash title="CentOS"
yum install -y keepalived
```

```bash title="Ubuntu"
apt-get -y install libssl-dev
apt-get -y install openssl
apt-get -y install libpopt-dev
apt-get -y install keepalived
```

## 部署 Keepalived

:::warning
注意！当前调用健康监测脚本内容为注释状态，原因是在安装 Rainbond 前 需要确保 VIP 已经存在；在 Rainbond 安装完成之后需将注释取消，才能实现健康监测，确保 网关高可用。
:::

### 主节点配置文件

```bash title="vi vi /etc/keepalived/keepalived.conf"

! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}
#vrrp_script check_gateway {
# 检测脚本
#   script "/etc/keepalived/check_gateway_status.sh"
# 执行间隔时间
#   interval 5
#}
vrrp_instance VI_1 {
    				
    #因使用非抢占模式，这里都为backup
    state BACKUP 
    #网卡设备名，通过 ifconfig 命令确定  
    interface ens6f0       
    virtual_router_id 51
    #优先级，主节点大于备节点     
    priority 100	
    advert_int 1
    #非抢占模式
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        <VIP>				
     }
#     track_script {
#    check_gateway
#   }
}
```

### 从节点配置文件

```bash title="vi vi /etc/keepalived/keepalived.conf"

! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}
#vrrp_script check_gateway {
# 检测脚本
#  script "/etc/keepalived/check_gateway_status.sh"
# 执行间隔时间
#  interval 5
#  }	
    vrrp_instance VI_1 {
    #因使用非抢占模式，这里都为backup
    state BACKUP 
    #网卡设备名，通过 ifconfig 命令确定   
    interface ens6f0
    virtual_router_id 51
    #优先级，主节点大于备节点   
    priority 50
    advert_int 1
    #非抢占模式
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        <VIP>			
    }
#     track_script {
#    check_gateway
#   }
}
```


### 健康监测脚本

扩展对网关节点健康检查的脚本，脚本的功能是当 rbd-gateway 组件停止服务，则关闭本机的 Keepalived，切换 VIP 。(主从都需操作)

```bash title="vi /etc/keepalived/check_gateway_status.sh"
#!/bin/bash
/usr/bin/curl -I http://localhost:10254/healthz

if [ $? -ne 0 ]; then
    cat /var/run/keepalived.pid | xargs kill
fi
```

添加执行权限

```bash
$ chmod +x /etc/keepalived/check_gateway_status.sh
```

### 修改 Keepalived Systemd 配置文件

更改 Keepalived systemd 配置文件，添加两项配置 `Restart` `RestartSec` ，保证 Keepalived 服务异常退出后自动重启。

```bash title="vi /lib/systemd/system/keepalived.service"
# /lib/systemd/system/keepalived.service
[Unit]
Description=Keepalive Daemon (LVS and VRRP)
After=network-online.target
Wants=network-online.target
# Only start if there is a configuration file
ConditionFileNotEmpty=/etc/keepalived/keepalived.conf
[Service]
Type=forking
KillMode=process
PIDFile=/var/run/keepalived.pid
# Read configuration variable file if it is present
EnvironmentFile=-/etc/default/keepalived
ExecStart=/usr/sbin/keepalived $DAEMON_ARGS
ExecReload=/bin/kill -HUP $MAINPID
# 总是重启该服务
Restart=always
# 重启间隔时间
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 启动 Keepalived 服务

启动服务并设置开机自启动

```bash
systemctl start keepalived
systemctl enable keepalived
systemctl status keepalived
```

### 检测 VIP 是否启动

```bash
ip a |grep <VIP>
```

:::warning
在进行下一步操作前请确保 VIP 已经存在，如果 VIP 不存在，请重新审查本节操作；在 Rainbond 安装完成之后请将配置文件中注释取消并重启 Keepalived 服务，实现健康监测，以此确保 网关高可用。
:::