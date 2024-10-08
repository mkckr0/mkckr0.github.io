---
title: git 省略 commit message
date: 2021-12-22 00:23
---

每次提交使用
```sh
git commit --allow-empty-message --no-edit
```

也可以设置命令别名
```sh
git config --global alias.nocommit "commit --allow-empty-message --no-edit"
git nocommit
```
