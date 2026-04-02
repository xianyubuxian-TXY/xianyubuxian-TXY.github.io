---
title: git的使用
date: 2026-03-22 10:45:01
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- git的使用
---

# 一、名称介绍

## 1.工作区、暂存区、版本库

### （1）工作区
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322105547093.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322110315732.png)

### （2）暂存区
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322105645503.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322110341439.png)

### （3）版本库
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322105717751.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322110410822.png)


## 2.分支
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322121355536.png)
### （1）Git 到底在保存
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120238377.png)

### （2）什么是分支
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120330057.png)

### （3）为什么需要分支
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120405713.png)

### （4）分支不是“拷贝”，而是“分叉”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120449872.png)

### （5）HEAD 是什么，它和分支是什么关系
- **HEAD 指向的是“当前所在分支”**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120549185.png)

### （6）创建分支时到底发生了什么
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120653152.png)

### （7）在不同分支提交，会发生什么
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120736824.png)

### （8）分支合并是什么
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120854664.png)

### （9）为什么有时候合并很简单，有时候会冲突
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322120924576.png)

### （10）分支和 rebase 的关系是什么
- **也可能发生冲突**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322121020880.png)

### （11）为什么说分支“很轻量”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322121121592.png)

### （12）什么时候一个分支算“死了”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322121311424.png)

## 3.分支冲突

### （1）什么是冲突
- **冲突**：Git 在**合并两个版本时**，发现同一个位置都被改了，但它不能确定该保留哪一份，所以需要你手动决定
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322114328715.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322115229666.png)

### （2）哪些操作容易产生冲突
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322114631859.png)

### （3）冲突出现后会看到什么
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322114722547.png)

### （4）处理冲突的标准流程
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322114822323.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322114846842.png)
- **辅助指令**：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322115048650.png)

### （5）冲突时的中止命令
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322115139362.png)

### （6）保留冲突指定一边内容
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322115345602.png)


## 4.PR 与 MR
### （1）PR、MR是什么？
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322123715182.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322123734190.png)

### （2）它们和直接本地 git merge 的区别
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322123832045.png)

### （3）为什么团队项目常用 PR / MR
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322123911885.png)

### （4）一个典型流程
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322123956525.png)

## 5.Fork仓库
### （1）Fork是什么
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322124242077.png)

### （2）为什么要Fork
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322124319781.png)

### （3）Fork 和 Clone 的区别
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322124414405.png)

# 二、Git指令

## 1.Git 基本操作
```bash
# 初始化仓库
git init

# 配置用户信息
git config --global user.name "Zhang San"
git config --global user.email "zhangsan@foo.com"

# 查看配置
git config --global --list

# 查看状态
git status

# 加入暂存区
git add 文件名
git add .

# 提交
git commit -m "说明"

# 查看日志
git log
git log --oneline	# 一行显示简洁历史

# 查看操作记录 ——> ⚠️ 即使 reset --hard 回退了，也通常可以通过 reflog 找回原来的版本
git reflog	

# 撤销工作区修改 ——> 恢复到暂存区状态（上一次add的状态）
git restore 文件名

# 取消暂存但保留修改	——> 已经 add 到暂存区了，但还没 commit，想取消暂存，保留修改
git restore --staged 文件名

# 回退到上一个版本
git reset --hard HEAD^

# 回退到指定版本
git reset --hard 版本号

# 查看差异
git diff HEAD -- 文件名			# 对比工作区和当前版本库中的文件

git diff HEAD HEAD^ -- 文件名	# 对比两个版本之间某个文件的差异

git diff HEAD origin/master		# 看本地和远程分支差异,用于fetch 之后检查远程更新

# 删除文件并纳入版本管理
git rm 文件名
git commit -m "删除文件"
```

## 2.Git 仓库管理指令
```bash
# 添加远程仓库 ——> 把远程仓库和本地仓库关联起来
# origin 是远程仓库的默认名字，可以自己改
# 后面是远程仓库地址，可以是 SSH 或 HTTP
git remote add origin git@github.com:username/repository.git

# 查看远程仓库 ——> 查看远程仓库名称和对应地址
git remote -v

# 修改远程仓库地址为 HTTP
git remote set-url origin https://github.com/username/repository.git

# 修改远程仓库地址为 SSH
git remote set-url origin git@github.com:username/repository.git

# HTTP 克隆
git clone https://github.com/username/repository.git

# SSH 克隆
git clone git@github.com:username/repository.git

# 第一次推送并关联分支 ——> 把本地分支推送到远程,建立本地分支和远程分支的跟踪关系
git push -u origin main

# 普通推送 ——> 把本地当前分支代码推送到远程仓库指定分支
git push origin main


# 拉取 ——> 等价于 fetch origin master + git merge origin/master
git pull origin master


# ========== 如果远程仓库比本地新，通常要先拉取再推送 ============
# 1.先抓取远程更新 ——> 可配合 git diff HEAD origin/master查看远程分支与本地分支的区别
git fetch origin master

# 2.再合并到本地
# origin/master ——> 远程仓库 origin 的 master 分支在本地的记录位置
git merge origin/master

# 3. 然后再推送
git push origin master
```
### 拓：什么时候更适合用 fetch + merge
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322113837683.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322113852795.png)
```bash
# 很多人团队开发时更习惯
git fetch origin
git diff HEAD origin/master
git merge origin/master
```

## 3.Git 分支管理相关指令
```bash
# 查看本地分支
git branch

# 查看远程分支
git branch -r

# 查看所有分支 ——> 同时查看本地分支和远程分支
git branch -a

# 创建分支 ——> 但不会自动切换过去
git branch dev

# 切换分支
git checkout dev
git switch dev	# 新写法，更清晰，专门用于切换分支

# 创建新分支并立即切换过去
git checkout -b dev
git switch -c dev

# 合并分支 ——> 把dev的内容合并到当前分支，通常先切换到main分支： git switch dev
git merge dev

# 删除分支 ——> 这个分支的修改通常已经被合并，否则 Git 会阻止删除
git branch -d dev

# 强制删除分支 ——> 强制删除本地分支，即使还没合并
git branch -D dev

# 重命名分支
git branch -m oldname newname

# 查看已经合并到当前分支的分支
git branch --merged

# 查看还没有合并到当前分支的分支 ——> 这个命令很适合删除分支前检查
git branch --no-merged

# 查看分支图 ——> 一行显示提交、图形化显示分支结构、展示所有分支历史
git log --oneline --graph --all

# 第一次推送并建立跟踪关系 ——> origin：远程仓库名  dev：本地分支
git push -u origin dev

# 推送分支到远程
git push origin dev

# 删除远程分支
git push origin --delete dev

# 基于远程分支创建本地分支
git switch -c dev origin/dev

# 变基
git rebase main

# rebase 继续
git rebase --continue

# rebase 放弃
git rebase --abort
```

## 拓展1：常见新功能开发
- 本地已经合并到 main 后，通常就是推送 main 到远程
- 功能分支推到远程，通常是为了评审、测试、协作，再决定是否合并
- 合并完成后，通常会删除本地和远程功能分支
- 团队项目里更常见的是：不直接本地合并后推主分支，而是推远程功能分支后走 PR/MR 流程
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322122333470.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322122632486.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322122728158.png)


## 拓展2：git rebase指令
- **核心作用**：把一条分支上的提交，“重新接”到另一条分支的最新提交后面。
- **可能会发生“冲突”**

```bash
# 当前分支变基到 master
git rebase master

# 基于远程主分支变基
git fetch origin
git rebase origin/master

# rebase 冲突后继续
git add 文件名
git rebase --continue

# 放弃 rebase
git rebase --abort

# 拉取远程时使用 rebase
git pull --rebase origin master
```

### （1）rebase是什么
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322115556713.png)

### （2）rebase 和 merge 的区别
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322115708362.png)

### （3）什么时候用 rebase
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322122954734.png)

### （4）rebase 冲突怎么处理
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322123114806.png)

### （5）pull --rebase 是什么
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322123254342.png)

# 三、如何创建新分支实现新功能，再合并
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260401171831154.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260401171846835.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260401171908747.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260401171920564.png)

# 四、如何参与开源项目

## 1.流程概述
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322124609620.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322124633053.png)

## 拓：Fork 后为什么还要加 upstream
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260322124817277.png)