---
title: Linux设置VPN代理（解决github连接问题）
date: 2026-01-12 16:50:52
tags:
	- 笔记
	- Linux
categories:
	- Linux
	- Linux的使用
	- Linux设置VPN代理（解决github连接问题）
---

## 1.前提要求：宿主机上需要有VPN

## 2.设置临时代理步骤
#### （1）（虚拟机）通过 ifconfig 命令查看虚拟机的IP，确定局域网网关，如下：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260112165452726.png)
- **网关为：192.168.36.1 （即“最后一位”为 1）**

#### （2）（宿主机）打开VPN连接，并开启“允许局域网连接”，如下：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260112165739343.png)

#### （3）（虚拟机）设置“临时代理”，并进行测试，如下：
- 以上面我的网关 192.168.36.1为例
```bash
# 假设代理端口是 7890（通常都是）
export http_proxy=http://192.168.36.1:7890
export https_proxy=http://192.168.36.1:7890

# 测试
curl -I https://github.com
```

**补充**：Clash for Windows 查看代理端口
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260112170039920.png)


### 3.备选方案
如果配置后还是不行，可以：**使用SSH 克隆（绕过 HTTPS）**
- 需要先配置 SSH key

```bash
git clone https://github.com/HowardHinnant/date.git # 失败

git clone git@github.com:HowardHinnant/date.git # 成功
```