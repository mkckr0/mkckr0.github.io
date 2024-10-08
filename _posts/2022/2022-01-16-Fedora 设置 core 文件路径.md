---
title: Fedora 设置 core 文件路径
date: 2022-01-16 23:07
---

1. `sudo vim /etc/sysctl.conf` 输入 `kernel.core_pattern=core.%p`
2. `sudo /lib/systemd/systemd-sysctl` 使修改生效
3. `cat /proc/sys/kernel/core_pattern` 查看是否生效