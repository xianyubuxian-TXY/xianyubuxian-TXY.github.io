---
title: sysctl 与 systemctl
date: 2026-01-13 10:48:42
tags:
	- 笔记
	- Linux
categories:
	- Linux
	- Linux系统工具
	- sysctl 与 systemctl
---

# 1.sysctl 与 systemctl
sysctl 和 systemctl 是 Linux 系统中功能完全不同的两个工具，核心区别在于**管理对象**和**作用场景**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260113105938893.png)

**作用层面**：
  - sysctl 作用于 **内核层**：
  	- 修改的是内核运行时的参数，影响整个系统的底层行为（比如网络协议栈、内存管理）。
  - systemctl 作用于 **用户层/服务层**：
  	- 管理的是用户态的服务进程，不涉及内核参数的调整。