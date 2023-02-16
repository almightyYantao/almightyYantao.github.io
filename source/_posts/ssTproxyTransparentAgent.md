---
title: ss-tproxy 透明代理的设置方法  
date: 2023-02-16  
tags: ['代理','透明代理','vpn','流量转发']  
---
<meta name="referrer" content="no-referrer" />

# 一、介绍
## 1、什么是透明代理
在正向代理中，一个软件如果想走 client 的代理服务，我们必须显式配置该软件，对该软件来说，有没有走代理是很明确的，大家都“心知肚明”。而透明代理则与正向代理相反，当我们设置好合适的防火墙规则（仅以 Linux 的 iptables 为例），我们将不再需要显式配置这些软件来让其经过代理或者不经过代理（直连），因为这些软件发出的流量会自动被 iptables 规则所处理，那些我们认为需要代理的流量，会被通过合适的方法发送到 client 进程，而那些我们不需要代理的流量，则直接放行（直连）。这个过程对于我们使用的软件来说是完全透明的，软件自身对其一无所知。这就叫做 **透明代理**。注意，所谓透明是对我们使用的软件透明，而非对 client、server 或目标网站透明，理解这一点非常重要。

## 2、透明代理的工作原理
在正向代理中，期望使用代理的软件会通过 http、socks5 协议与 client 进程进行交互，以此完成代理操作。而在透明代理中，我们的软件发出的流量是完全正常的流量，并没有像正向代理那样，使用 http、socks5 等专用协议，这些流量经过 iptables 规则的处理后，会被通过“合适的方法”发送给 client 进程（当然是指那些我们认为需要走代理的流量）。注意，此时 client 进程接收到不再是 http、socks5 协议数据，而是经过 iptables 处理的“透明代理数据”，“透明代理数据”从本质上来说与正常数据没有区别，只是多了一些“元数据”在里面，使得 client 进程可以通过 netfilter 或操作系统提供的 API 接口来获取这些元数据（元数据其实就是原始目的地址和原始目的端口）。那么这个“合适的方法”是什么？目前来说有两种：

-   REDIRECT：只支持 TCP 协议的透明代理。
-   TPROXY：支持 TCP 和 UDP 协议的透明代理。

因此，对于 TCP 透明代理，有两种实现方式，一种是 REDIRECT，一种是 TPROXY；而对于 UDP 透明代理，只能通过 TPROXY 方式来实现。为方便叙述，本文以 **纯 TPROXY 模式** 指代 TCP 和 UDP 都使用 TPROXY 来实现，以 **REDIRECT + TPROXY 模式** 指代 TCP 使用 REDIRECT 实现，而 UDP 使用 TPROXY 来实现，有时候简称 **REDIRECT 模式**，它们都是一个意思。

# 二、Openvpn流量转发
采用：[https://github.com/zfl9/ss-tproxy](https://github.com/zfl9/ss-tproxy)
原理：openvpn → iptables → ss-redir

# 三、部署方式
## 1、依赖检查
在部署之前，需要检查下当前自己的服务器依赖是否完整：
依赖安装地址：[https://github.com/zfl9/ss-tproxy/wiki/Linux-%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86#%E5%AE%89%E8%A3%85%E4%BE%9D%E8%B5%96](https://github.com/zfl9/ss-tproxy/wiki/Linux-%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86#%E5%AE%89%E8%A3%85%E4%BE%9D%E8%B5%96)

## 2、脚本安装
```bash
git clone https:``//github.com/zfl9/ss-tproxy
cd ss-tproxy
chmod +x ss-tproxy
```
### install 可用的情况下（推荐）
```bash
install ss-tproxy /usr/local/bin
install -d /etc/ss-tproxy
install -m` `644` `ss-tproxy.conf gfwlist* chnroute* ignlist* /etc/ss-tproxy
install -m` `644` `ss-tproxy.service /etc/systemd/system # 可选，安装 service 文件
```
### install 不可用
```bash
cp -af ss-tproxy /usr/local/bin
mkdir -p /etc/ss-tproxy
cp -af ss-tproxy.conf gfwlist* chnroute* ignlist* /etc/ss-tproxy
cp -af ss-tproxy.service /etc/systemd/system # 可选，安装 service 文件
```

配置文件：
```bash
**/etc/ss-tproxy/ss-tproxy.conf**

## mode
#mode='global'  # global 模式 (不分流)
mode='gfwlist' # gfwlist 模式 (黑名单)
#mode='chnroute' # chnroute 模式 (白名单)
 
## ipv4/6
ipv4='true'     # true:启用ipv4透明代理; false:关闭ipv4透明代理
ipv6='false'    # true:启用ipv6透明代理; false:关闭ipv6透明代理
 
## tproxy
tproxy='false'  # true:TPROXY+TPROXY; false:REDIRECT+TPROXY
 
## tcponly
tcponly='false' # true:仅代理TCP流量; false:代理TCP和UDP流量
 
## selfonly
selfonly='false' # true:仅代理本机流量; false:代理本机及"内网"流量
 
## proxy
# user/group(#1,推荐) vs svraddr+port(#2), user/group选其中一个填写(不建议都填)
proxy_procuser='proxy'   # 本机代理进程的 user/uid，用来放行本机代理进程传出的流量
proxy_procgroup=''       # 本机代理进程的 group/gid，用来放行本机代理进程传出的流量
proxy_svraddr4=()        # 服务器的 IPv4 地址或域名，允许填写多个服务器地址，空格隔开
proxy_svraddr6=()        # 服务器的 IPv6 地址或域名，允许填写多个服务器地址，空格隔开
proxy_svrport=''         # 服务器的监听端口，可填多个端口，格式同 ipts_proxy_dst_port
proxy_tcpport='12121'    # ss/ssr/v2ray 等本机进程的 TCP 监听端口，该端口支持透明代理
proxy_udpport='12121'    # ss/ssr/v2ray 等本机进程的 UDP 监听端口，该端口支持透明代理
proxy_startcmd='(ss-redir -c /etc/shadowsocks-libev/config.json -u </dev/null &>>/var/log/ss-redir.log &)'     # 用于启动本机代理进程的 shell 命令，该命令应该能立即执行完毕
proxy_stopcmd='kill -9 $(pidof ss-redir)'      # 用于关闭本机代理进程的 shell 命令，该命令应该能立即执行完毕
 
## dns
dns_direct='223.5.5.5'                # 本地 IPv4 DNS，不能指定端口，也可以填组织、公司内部 DNS
dns_direct6='240C::6666'              # 本地 IPv6 DNS，不能指定端口，也可以填组织、公司内部 DNS
dns_remote='8.8.8.8#53'               # 远程 IPv4 DNS，必须指定端口，提示：访问远程 DNS 会走代理
dns_remote6='2001:4860:4860::8888#53' # 远程 IPv6 DNS，必须指定端口，提示：访问远程 DNS 会走代理
 
## dnsmasq
dnsmasq_bind_port='53'                  # dnsmasq 服务器监听端口，见 README
dnsmasq_cache_size='4096'               # DNS 缓存大小，大小为 0 表示禁用缓存
dnsmasq_cache_time='3600'               # DNS 缓存时间，单位是秒，最大 3600 秒
dnsmasq_query_maxcnt='1024'             # 设置并发 DNS 查询的最大数量，默认为 150
dnsmasq_log_enable='false'              # 记录详细日志，除非进行调试，否则不建议启用
dnsmasq_log_file='/var/log/dnsmasq.log' # 日志文件，如果不想保存日志可以改为 /dev/null
dnsmasq_conf_dir=()                     # `--conf-dir` 选项的参数，可以填多个，空格隔开
dnsmasq_conf_file=()                    # `--conf-file` 选项的参数，可以填多个，空格隔开
dnsmasq_conf_string=()                  # 自定义配置，一个数组元素就是一行配置，空格隔开
 
## chinadns
chinadns_bind_port='65353'               # chinadns-ng 服务器监听端口，通常不用改动
chinadns_timeout='5'                     # 等待上游 DNS 返回响应的超时时间，单位为秒
chinadns_repeat='1'                      # 向可信 DNS 发送几次 DNS 查询请求，默认为 1
chinadns_fairmode='true'                 # 使用公平模式，具体看 chinadns-ng 的 README
chinadns_gfwlist_mode='true'             # gfwlist 模式，加载 gfwlist.txt/gfwlist.ext
chinadns_noip_as_chnip='false'           # 启用 chinadns-ng 的 `--noip-as-chnip` 选项
chinadns_verbose='false'                 # 记录详细日志，除非进行调试，否则不建议启用
chinadns_logfile='/var/log/chinadns.log' # 日志文件，如果不想保存日志可以改为 /dev/null
chinadns_privaddr4=()                    # IPv4 私有地址段，多个用空格隔开，具体见 README
chinadns_privaddr6=()                    # IPv6 私有地址段，多个用空格隔开，具体见 README
 
## dns2tcp
dns2tcp_bind_port='65454'               # dns2tcp 转发服务器监听端口，如有冲突请修改
dns2tcp_tcp_syncnt=''                   # dns2tcp 的 `-s` 选项，留空表示不设置此选项
dns2tcp_tcp_quickack='false'            # dns2tcp 的 `-a` 选项，选项取值为true/false
dns2tcp_tcp_fastopen='false'            # dns2tcp 的 `-f` 选项，选项取值为true/false
dns2tcp_verbose='false'                 # 记录详细日志，除非进行调试，否则不建议启用
dns2tcp_logfile='/var/log/dns2tcp.log'  # 日志文件，如果不想保存日志可以改为 /dev/null
 
## ipts
ipts_if_lo='lo'                 # 环回接口的名称，在标准发行版中，通常为 lo，如果不是请修改
ipts_rt_tab='233'               # iproute2 路由表名或表 ID，除非产生冲突，否则不建议改动该选项
ipts_rt_mark='0x2333'           # iproute2 策略路由的防火墙标记，除非产生冲突，否则不建议改动该选项
ipts_set_snat='true'           # 设置 iptables 的 MASQUERADE 规则，布尔值，`true/false`，详见 README
ipts_set_snat6='false'          # 设置 ip6tables 的 MASQUERADE 规则，布尔值，`true/false`，详见 README
ipts_reddns_onstop='true'       # ss-tproxy stop 后，是否将其它主机发至本机的 DNS 重定向至直连 DNS，详见 README
ipts_proxy_dst_port='1:65535'   # 黑名单 IP 的哪些端口走代理，多个用逗号隔开，冒号为端口范围(含边界)，详见 README
 
## opts
opts_ss_netstat='auto'                  # auto/ss/netstat，用哪个端口检测工具，见 README
opts_ping_cmd_to_use='auto'             # auto/standalone/parameter，ping 相关，见 README
opts_hostname_resolver='auto'           # auto/dig/getent/ping，用哪个解析工具，见 README
opts_overwrite_resolv='false'           # true/false/留空，如何操作 resolv.conf，见 README
opts_ip_for_check_net='223.5.5.5'       # 检测外网是否可访问的 IP，ping，留空表示跳过此检查
 
## file
file_gfwlist_txt='/etc/ss-tproxy/gfwlist.txt'      # gfwlist/chnlist 模式预置文件
file_gfwlist_ext='/etc/ss-tproxy/gfwlist.ext'      # gfwlist/chnlist 模式扩展文件
file_ignlist_ext='/etc/ss-tproxy/ignlist.ext'      # global/chnroute 模式扩展文件
file_chnroute_set='/etc/ss-tproxy/chnroute.set'    # chnroute 地址段文件 (iptables)
file_chnroute6_set='/etc/ss-tproxy/chnroute6.set'  # chnroute6 地址段文件 (ip6tables)
file_dnsserver_pid='/etc/ss-tproxy/.dnsserver.pid' # dns 服务器进程的 pid 文件 (shell)
 
## 主要放通下内网的访问，然后把MASQUERADE提前出来，要不然tcp的连接会回不来
post_start(){
iptables -t nat -I PREROUTING -s 10.207.0.0/16 -d 10.0.0.0/8 -j ACCEPT
iptables -t nat -I SSTP_POSTROUTING -s 10.207.0.0/16 -j MASQUERADE
}
 
post_stop(){
iptables -t nat -D PREROUTING -s 10.207.0.0/16 -d 10.0.0.0/8 -j ACCEPT
iptables -t nat -D SSTP_POSTROUTING -s 10.207.0.0/16 -j MASQUERADE
}
```

以下配置需要特别注意，如果不知道如何配置的，那么就按照我这么默认配置即可
具体配置介绍：[https://github.com/zfl9/ss-tproxy#%E9%85%8D%E7%BD%AE%E8%AF%B4%E6%98%8E](https://github.com/zfl9/ss-tproxy#%E9%85%8D%E7%BD%AE%E8%AF%B4%E6%98%8E)
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/20230216143840.png"/>
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/20230216142810.png" />

## 3、设置ss-libev代理
```bash
#新版本(v4.``6.1``及以上)
#第一次运行时，请执行下面这两个操作
#``1``.创建proxy用户和组: useradd -Mr -d/tmp -s/bin/bash proxy
#``2``.授予透明代理相关权限: setcap cap_net_bind_service,cap_net_admin+ep /path/to/ss-redir
#>> 若setcap不可用，可使用suid权限位，此时需配置：proxy_procuser=``''``、proxy_procgroup=``'proxy'
#>> 将所有者(组)改为root，并授予suid权限：chown root:root /path/to/ss-redir && chmod` `4755` `/path/to/ss-redir
proxy_procuser=``'proxy'
#proxy_startcmd=``'su proxy -c"(ss-redir -c /etc/ss.json -u -v </dev/null &>>/tmp/ss-redir.log &)"'` `# -v 表示记录详细日志
proxy_startcmd=``'su proxy -c"(ss-redir -c /etc/ss.json -u </dev/null &>>/tmp/ss-redir.log &)"'` `# 这里就不记录详细日志了
proxy_stopcmd=``'kill -9 $(pidof ss-redir)'
```
## 黑名单、白名单说明
-   对于 global 模式，白名单文件为 `ignlist.ext`，没有黑名单文件，因为默认都走代理。
-   对于 gfwlist 模式，黑名单文件为 `gfwlist.txt/ext`，没有白名单文件，因为其它都走直连。
-   对于 chnroute 模式，白名单文件为 `ignlist.ext`，没有黑名单文件，但允许开启此功能，见下。
如果想让 chnroute 模式支持黑名单扩展，请打开 chinadns-ng 的 gfwlist 模式（`chinadns_gfwlist_mode`）；开启 gfwlist 模式后，chinadns-ng 会读取 `gfwlist.txt/ext` 黑名单文件中的**域名模式**；当 chinadns-ng 收到域名解析请求时，会先检查给定域名是否在黑名单中，如果是则只向可信 DNS 发出解析请求（也就是 `dns_remote/dns_remote6`），因此解析出来的会是国外 IP（不一定，具体要看给定域名的A/AAAA记录以及其dns解析设定），然后当客户端访问该 IP 时就会走代理出去了（如果解析的地址是国外地址）。

> `chinadns_gfwlist_mode`的本意其实并不是为了支持'黑名单'，而是为了提高 chinadns-ng 的准确性，降低 dns 污染的可能性


启动后，ss-tproxy会自动帮你做透明代理，会设置好iptables的配置，不需要在手工配置
同时ss-tproxy还提供勾子的形式，来帮助解决分流或者内网访问的一个方案，具体参考：
-   [https://github.com/zfl9/ss-tproxy#%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0](https://github.com/zfl9/ss-tproxy#%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0)
-   [https://github.com/zfl9/ss-tproxy#%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0%E5%B0%8F%E6%8A%80%E5%B7%A7](https://github.com/zfl9/ss-tproxy#%E9%92%A9%E5%AD%90%E5%87%BD%E6%95%B0%E5%B0%8F%E6%8A%80%E5%B7%A7)