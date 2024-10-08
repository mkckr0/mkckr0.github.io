---
title: VirtualBox 同时添加 NAT 和 Host-Only 网卡出现无法上网的情况
date: 2021-12-22 07:29
---

如果网卡1是 NAT，网卡2是 Host-Only，可以 ping 通 baidu.com。  
如果网卡1是 Host-Only，网卡2是 NAT，无法 ping 通 baidu.com。

使用 nmcli 修改 NAT 网卡和 Host-Only 网卡的 ipv4.route-metric，分别设置为 1 和 2。这样 NAT 网卡的路由优先级比 Host-Only 网卡更高，可以 ping 通 baidu.com。
