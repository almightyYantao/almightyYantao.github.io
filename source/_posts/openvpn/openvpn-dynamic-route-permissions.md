---
title: openvpn动态下发权限
date: 2022-12-14  
tags: ['openvpn','权限','动态下发']  
index_img: "/img/openvpn.webp"
---

## 首先了解我们为什么要动态下发
- 开源的openvpn并不支持权限管理，大部分在做权限管理的时候使用的都是根据来源IP或IP段通过iptables/交换机来进行权限控制
- 权限控制太广了，根本无法很好的去做管理后台的配置，特别是需要用户组来进行区分的时候，那就更困难了

## 权限下发的逻辑

{% plantuml %}  
title openvpn连接示意图
用户 -> openvpn:通过公网连接openvpn
openvpn->openvpn:下发IP地址，设置动态权限
openvpn->用户:完成连接
{% endplantuml %}

## 用到的技术
- ipset
- iptables

## 主要脚本内容
### 连接脚本

```bash
/sbin/ipset create ${common_name}-${common_ip} hash:ip
/sbin/ipset create ${common_name}-${common_ip}-drop hash:ip
# 这里是动态去你的后端获取出来的允许访问的列表，接口自己去实现
for ((i = 0; i < ${#permissionsAccept[@]}; i++)); do
    /sbin/ipset add ${common_name}-${common_ip} ${permissionsAccept[$i]//\"/}
done

for ((i = 0; i < ${#permissionsDrop[@]}; i++)); do
    /sbin/ipset add ${common_name}-${common_ip}-drop ${permissionsDrop[$i]//\"/}
done
# 设置iptables
/sbin/iptables -A FORWARD -s $common_ip -m set --match-set ${common_name}-${common_ip} dst -j ACCEPT
/sbin/iptables -A FORWARD -s $common_ip -m set --match-set ${common_name}-${common_ip}-drop dst -j DROP
/sbin/iptables -A FORWARD -s $common_ip -j DROP
```

### 断开脚本

```bash
/sbin/iptables -D FORWARD -s $ifconfig_pool_remote_ip -m set --match-set ${common_name}-${ifconfig_pool_remote_ip} dst -j ACCEPT
/sbin/iptables -D FORWARD -s $ifconfig_pool_remote_ip -m set --match-set ${common_name}-${ifconfig_pool_remote_ip}-drop dst -j DROP
/sbin/iptables -D FORWARD -s $ifconfig_pool_remote_ip -j DROP
/sbin/ipset destroy ${common_name}-${ifconfig_pool_remote_ip}
/sbin/ipset destroy ${common_name}-${ifconfig_pool_remote_ip}-drop
```

## 如果在运行的过程中出现脚本权限不足
```bash
chmod 766 connect.sh disconnect.sh
chmod a+x connect.sh disconnect.sh
chmod +s connect.sh disconnect.sh
```