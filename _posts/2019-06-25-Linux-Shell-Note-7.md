---
layout:     post
title:      "Linux Shell Note 7"
subtitle:   "Network"
date:       2019-06-25
author:     ChuRiver
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Linux
    - Shell
---

本篇将网络相关命令

## index
- [网络配置](#网络配置)
- [ping](#ping)
- [列出所有活动主机(shell)](#列出所有活动主机shell)
- [fping](#fping)
- [并行ping(shell)](#并行pingshell)
- [网络传输文件](#网络传输文件)
- [连接无线网络](#连接无线网络)
- [SSH](#ssh)
- [SFTP](#sftp)
- [SCP](#scp)
- [无密码SSH自动登录](#无密码ssh自动登录)
- [端口转发](#端口转发)
- [本地挂载远程驱动器](#本地挂载远程驱动器)
- [网络流量与端口分析](#网络流量与端口分析)
- [创建套接字](#创建套接字)
- [互联网连接共享](#互联网连接共享)
- [iptables防火墙](#iptables防火墙)

## 网络配置

查看网络状态命令：`ifconfig`

手动设置ip地址
```bash
ifconfig wlan0 192.168.0.80
ifconfig wlan0 192.168.0.80 netmask 255.255.252.0
```

主动获取dhcp地址
```bash
dhclient eth0
```

打印网络接口列表
```bash
iconfig | cut -c -10 | tr -d ' ' | tr -s '\n'
```

显示ip地址
```bash
ifconfig iface_name
```

提取ip地址
```bash
ifconfig iface_name | egrep -o "inet addr:[^ ]*" | grep -o "[0-9.]*"
```

硬件地址(MAC)指定
```bash
ifconfig eth0 hw ether 00:12:34:56:78:90
```

dns服务器地址存储在文件`/etc/resolv.conf`内部，可进行增删查改等操作。

dns解析域名
```bash
host google.com
nslookup google.com
```

本地DNS解析指定存储在文件`/etc/hosts`内，可修改此文件实现指定的DNS解析结果。

显示路由表信息
```bash
route
route -n # 以数字形式表示地址
```

设置默认网关
```bash
route add default gw IP_ADDRESS INTERFACE_NAME
```

## ping

使用ICMP协议查看目标主机是否存活：`ping address`

限制发送组数
```bash
ping -c COUNT address
```

按每一跳地址追踪ip连接之间经过的路由器节点：
```bash
traceroute google.com
```

## 列出所有活动主机(shell)
通过遍历局域网内所有主机，并使用ping判断是否存活。
```bash
#!/bin/bash
for ip in 192.168.0.{1..255};
do
    ping $ip -c 2 &> /dev/null;

    if [ $? -eq 0 ];
    then
        echo $ip is alive
    fi
done
```

## fping
专用ping查询存活主机命令：
```bash
fping -a 192.168.1.1/24 -g  > /dev/null
fping -a 192.168.0.1 192.168.0.255 -g

# -a 指定打印出所有活动的ip地址
# -u 指定打印出所有无法到达的主机
# -g 指定从“ip地址/子网掩码”或IP地址范围方式中生成一组IP地址
```

## 并行ping(shell)
后台指定单个`ping`任务，实现后台并发多`ping`
```bash
#!/bin/bash
for ip in 192.168.0.{1..255};
do`
    (
        ping $ip -c 2 &> /dev/null ;

        if [ $? -re 0 ];
        then
            echo $ip is alive
        fi
    )&
done
wait
```


## 网络传输文件
使用ftp协议
```bash
lftp user@host
get file
put file
quit
```

FTP传输脚本
```bash
#!/bin/bash
HOST='domain.com'
USER='foo'
PASSWD='password'
ftp -i -n $HOST <<EOF
user ${USER} ${PASSWD}
binary
cd /home/slynux
put testfile.jpg
get serverfile.jpg
quit
EOF
```


## 连接无线网络
```bash
#!/bin/bash
IFACE=wlan0
IP_ADDR=192.168.1.5
SUBNET_MASK=255.255.255.0
GW=192.168.1.1
HW_ADDR='00:12:34:56:78:90'
ESSID="homenet"
WEP_KEY=8b140b20e7
FREQ=2.462G
KEY_PART=""
if [[ -n $WEP_KEY ]];
then
    KEY_PART="key $WEP_KEY"
fi
/sbin/ifconfig $IFACE down
if [ $UID -ne 0 ];
then
    echo "Run as root"
    exit 1;
fi
if [[ -n $HW_ADDR ]];
then
    /sbin/ifconfig $IFACE hw ether $HW_ADDR
    echo Spoofed MAC ADDERSS to $HW_ADDR
fi
/sbin/iwconfig $IFACE essid $ESSID $KEY_PART freq $FREQ
/sbin/ifconfig $IFACE $IP_ADDR netmask $SUBNET_MASK
route add default gw $GW $IFACE
echo Successfully configured $IFACE
```


## SSH
连接远程主机
```bash
ssh username@remote_host
ssh user@localhost -p 422
```

执行命令
```bash
ssh user@host "COOMAND"
ssh user@host "command1; command2; command3"
```

查看每个远程主机登录用户
```bash
#!/bin/bash
IP_LIST="192.168.0.1 192.168.0.5"
USER="test"
for IP in $IP_LIST;
do
    utime=$(ssh ${USER}@${IP} uptime | awk '{ print $3 }' )
    echo $IP uptime: $utime
done
```

将传输数据压缩
```bash
ssh -C user@host COMMANDS
```

重定向
```bash
echo 'text' | ssh user@host 'echo'
ssh user@host 'echo' < file
```

执行图形化命令
```bash
ssh user@host "export DISPLAY=:0; command1; command2"""
ssh -X user@host "command1; command2"
```


## SFTP
基于ssh协议的ftp
```bash
sftp user@host
sftp -oPort=422 user@host
```

## SCP
基于ssh的数据传输
```bash
scp filename user@host:/home/path
scp SOURCE DESTINATION
scp user@host:/home/path/filename filename
# 也可使用-oPort
scp -r /home/slynux user@host:/home/backups
```


## 无密码SSH自动登录
配置公私钥
```
ssh user@host "cat >> ~/.ssh/authorized_keys" < ~/.ssh/id_rsa.pub
ssh-copy-id user@host
```


## 端口转发
本地至外部转发
```bash
ssh -L 8000:www.kernel.org:80 user@host
ssh -L 8000:www.kernel.org:80 user@remote_host
```

非交互式端口转发
```bash
ssh -fL 8000:www.kernel.org:80 user@host -N
# -f 后台执行
# -L 登录主机
# -N 无需执行命令
```

反向端口转发
```bash
ssh -R 8000:localhost:80 user@host
```


## 本地挂载远程驱动器
```bash
sshfs -o allow_other user@host:/home/path /mnt/mountpoint
umount /mnt/mountpoint
```


## 网络流量与端口分析
列出端口和服务
```bash
lsof -i
```

列出本地开放端口
```bash
lsof -i | grep ":[0-9]\+->" -o | grep "[0-9]\+" -o | sort + uniq
```

查看开放端口与服务
```bash
netstat -tnp
```

## 创建套接字
设置侦听套接字
```bash
nc -l 1234
```

连接
```bash
nc HOST 1234
```


## 互联网连接共享
```bash
#!/bin/bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -A FORWARD -i $1 -o $2 -s 10.99.0.0/16 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELEATED -j ACCEPT
iptables -A POSTROUTING -t nat -j MASQUERADE
```

使用：
```bash
./netsharing.sh eth0 wlan0
```


## iptables防火墙
阻塞指定ip
```bash
iptables -A OUTPUT -d 8.8.8.8 -j DROP
```

阻塞端口
```bash
iptables -A OUTPUT -p tcp -dport 21 -j DROP
```

清除所有规则
```bash
iptables --flush
```
