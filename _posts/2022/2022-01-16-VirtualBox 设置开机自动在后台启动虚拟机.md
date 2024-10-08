---
title: VirtualBox 设置开机自动在后台启动虚拟机
date: 2022-01-16 22:12
---

1. 打开 `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp`
2. 新建文件 `virtualbox.bat`
3. 编写脚本 `"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" startvm "os_0" --type headless`，其中 `os_0` 是虚拟机的名字。

还有一种改注册表的方法
使用 Powershell
```ps
reg add HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run /v VirtualBox /t REG_SZ /d '\"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe\" startvm \"os_0\" --type headless' /f
```