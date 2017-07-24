---
title: Git使用总结
date: 2016-12-22 20:10:50
tags: [git]
---

## 常用命令

```bash
git clone
git config
git checkout
git status
git add
git commit
git branch
git push
git pull
git log
git tag
.gitignore

```

### 1. git 
```bash
git pull
git status
git add filesxxxx; git add --all; git add dirs
git commit -m "change log"
git push

```

* git reset --soft  id
把当前本地commit的版本会退到某个版本，本地修改不会丢失

* git reset --hard id
本地修改会丢失。

* git push -u origin master
把当前本地分支代码推送到远端master分支上


图解git: http://marklodato.github.io/visual-git-guide/index-zh-cn.html




