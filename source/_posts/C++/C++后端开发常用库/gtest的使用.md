---
title: gtest的使用
date: 2026-01-15 10:12:44
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- gtest
---


# 1.常用命令
##### 1.在命令行指定运行要运行的测试用例
- 通过 --gtest_filter 参数指定要运行的测试
```bash
./bin/tests/config_test --gtest_filter="测试套件名.测试用例名"

# 例1：运行单个测试用例
# 运行 ConfigTest 套件中的 InvalidYamlFormat 测试
./bin/tests/config_test --gtest_filter=ConfigTest.InvalidYamlFormat

# 例2：运行多个测试用例（使用冒号分隔）
# 运行多个指定的测试
./bin/tests/config_test --gtest_filter=ConfigTest.InvalidYamlFormat:ConfigTest.ValidConfig

# 例3：运行整个测试套件
# 运行 ConfigTest 套件中的所有测试
./bin/tests/config_test --gtest_filter=ConfigTest.*
```
