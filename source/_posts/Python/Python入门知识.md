---
title: Python入门知识
date: 2026-01-09 20:22:24
tags:
	- 笔记
categories:
	- Python
---

# 一、基础类型
## 1.数字
### （1）数字类型总览
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109210235954.png)

### （2）运算符
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109211802890.png)

### （3）关键特性
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109210634283.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109210655826.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109210730504.png)
#### 补充："不可变性"
- **所有数字类型都是不可变对象（修改会生成新对象）**
```python
# 地址唯一标识对象

# 1. 整数示例
a = 10
print(f"修改前 a 的值：{a}，内存地址：{id(a)}") # 修改前 a 的值：10，内存地址：140708467500096  

# 看似“修改”a，实际是创建新对象
a += 5
print(f"修改后 a 的值：{a}，内存地址：{id(a)}")  # 修改后 a 的值：15，内存地址：140708467500256

# 2. 浮点数示例
b = 3.14
print(f"\n修改前 b 的值：{b}，内存地址：{id(b)}") # 修改前 b 的值：3.14，内存地址：2865657515952

b = b * 2
print(f"修改后 b 的值：{b}，内存地址：{id(b)}")  # 修改后 b 的值：6.28，内存地址：2865657516176

# 3. 复数示例
c = 1 + 2j
print(f"\n修改前 c 的值：{c}，内存地址：{id(c)}") # 修改前 c 的值：(1+2j)，内存地址：2865657468880

c = c + (3 + 4j)
print(f"修改后 c 的值：{c}，内存地址：{id(c)}")  # 修改后 c 的值：(4+6j)，内存地址：2865657469008
```


### （4）类型转换
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109211200538.png)

### （5）常用内置函数
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109211352194.png)


## 2.bool(布尔)类型
### （1）bool 类型的基础定义与本质
- **bool类型的定义**： Python 中用于**表示「真/假」逻辑判断**的基础类型，核心取值仅两个：**True（真）** 和 **False（假）**
- **bool类型的本质**：特殊的整数，其中 True 等价于整数 1，False 等价于整数 0

### （2）bool 类型的核心特性
- **取值唯一**：某个布尔表达式的结果在特定时刻只能是True或False之一
	- **首字母必须大写**，true、false为无效类型
- **不可变性**：与数字、字符串类型一致，bool 类型也是不可变对象，一旦创建无法修改值（但因取值唯一，实际无修改场景）
- **子类特性**：bool 是 int 的子类，因此可继承 int 的部分属性和方法，但自身无额外特有方法；
- **逻辑运算核心**：支持and（与）、or（或）、not（非）三种核心逻辑运算
	- **注意**：不是 & |  ~

### （3）其他类型到 bool 类型的转换规则
- **Python 中，任何类型的对象都可以通过 bool() 函数转为 bool 类型** 
	- **转换规则**：空/零为 False，非空/非零为 True
	![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109222801836.png)
- **万物皆bool**：所有类型都可以直接作为bool参与判断（**隐式转换**）
```python
# 1. 数值类型：0/0.0/0+0j 为 False，其他为 True
if 10:  # 非零 → True
    print("10 是 True")

if 0.0:  # 零 → False（不执行）
    print("0.0 是 True")

# 2. 字符串：空字符串 '' 为 False，其他（含空白）为 True
if "hello":  # 非空 → True
    print("非空字符串是 True")

if "   ":  # 空白字符串（非空）→ True
    print("空白字符串也是 True")

if "":  # 空字符串 → False（不执行）
    print("空字符串是 True")

# 3. 容器类型：空列表/元组/字典为 False，非空为 True
if [1,2,3]:  # 非空列表 → True
    print("非空列表是 True")

if {}:  # 空字典 → False（不执行）
    print("空字典是 True")

# 4. None：直接判断，无需写 == None
x = None
if not x:  # None → False，not 后为 True
    print("x 是 None")
```

## 3.字符串
### （1）字符串的定义与基础表示
- **单引号（'）**：适用于短文本，内部可直接包含双引号（无需转义）
- **双引号（"）**：与单引号功能基本一致，内部可直接包含单引号
- **三引号（''' 或 """）**：支持多行文本，可保留换行、空格等格式，也可用于文档字符串（注释）
- **原始字符串（r/R 前缀）**：禁止转义字符（\）生效，适用于路径、正则表达式等场景
```python
# 1. 单引号与双引号
str1 = 'Hello Python'
str2 = "Hello 'Python'"  # 内部包含单引号，用双引号包裹无需转义
str3 = 'Hello "Python"'  # 内部包含双引号，用单引号包裹无需转义

# 2. 三引号（多行文本）
str4 = '''第一行文本
第二行文本
第三行文本'''  # 保留换行格式

# 3. 原始字符串（禁止转义）
str5 = r'C:\Users\test\Desktop'  # 无需写成 C:\\Users\\test\\Desktop
str6 = R'hello\nworld'  # 输出：hello\nworld（\n不被解析为换行）

print(str2)  # Hello 'Python'
print(str4)  # 按多行格式输出
print(str5)  # C:\Users\test\Desktop
```

### （2）字符串的常用操作
#### <1>拼接与重复
- **拼接**：用 + 运算符，将多个字符串合并为一个新字符串
- **重复**：用 * 运算符，将字符串重复指定次数
```python
# 拼接
s1 = 'Hello'
s2 = 'World'
s3 = s1 + ', ' + s2 + '!'
print(s3)  # Hello, World!

# 重复
s4 = 'ab' * 3
print(s4)  # ababab

# 注意：字符串不能与非字符串类型直接拼接，需先转换类型
num = 123
# s5 = s1 + num  # 报错：TypeError: can only concatenate str (not "int") to str
s5 = s1 + str(num)  # 先将int转为str
print(s5)  # Hello123
```

#### <2>格式化转换
**字符串格式化用于将变量/表达式的值插入到字符串中，常用三种方式：**
- **% 格式化**：传统方式，类似 C 语言的 printf；
- **str.format()**：Python 2.6+ 引入，更灵活的格式化方式；
- **f-string（f/F 前缀）**：Python 3.6+ 引入，最简洁高效的格式化方式，支持在字符串中**直接嵌入变量/表达式**。（**最推荐！**）
```python
name = '小明'
age = 20
score = 95.5

# 1. % 格式化
print('姓名：%s，年龄：%d，成绩：%.1f' % (name, age, score))  # 姓名：小明，年龄：20，成绩：95.5
# 说明：%s（字符串）、%d（整数）、%.1f（保留1位小数的浮点数）

# 2. str.format() 格式化
print('姓名：{}，年龄：{}，成绩：{:.1f}'.format(name, age, score))  # 同上
print('姓名：{0}，年龄：{1}，成绩：{2:.1f}'.format(name, age, score))  # 按索引指定参数
print('姓名：{n}，年龄：{a}，成绩：{s:.1f}'.format(n=name, a=age, s=score))  # 按参数名指定

# 3. f-string 格式化（推荐）
print(f'姓名：{name}，年龄：{age}，成绩：{score:.1f}')  # 同上
print(f'明年年龄：{age + 1}')  # 支持表达式：明年年龄：21
print(f'成绩的平方：{score ** 2}')  # 支持表达式：成绩的平方：9120.25
```

### （3）字符串常用内置函数与方法
#### <1>内置函数（全局函数，可直接调用）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109214950265.png)

#### <2>字符串对象方法（需通过字符串实例调用，s.方法名()）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109215553043.png)

### （4）字符串的核心特性
#### <1>不可变性
- 字符串对象**一旦创建**，其内部的字符序列就**无法被修改**（包括增删、替换单个字符）
```python
s = 'hello'
# s[0] = 'H'  # 报错：TypeError: 'str' object does not support item assignment（字符串不支持元素赋值）

# 若需修改，需通过切片、拼接生成新字符串
s_new = 'H' + s[1:]  # 拼接：用'H'替换第一个字符
print(s_new)  # Hello
```
- 任何看似 “修改” 字符串的操作，本质都是**创建一个全新的字符串对象**，原对象保持不变。
```python
# 初始字符串
s = "hello"
print(f"初始值：{s}，内存地址：{id(s)}")  # 比如：初始值：hello，地址：140708467898160

# 1. 拼接操作（看似修改，实际新建）
s += " world"
print(f"拼接后：{s}，内存地址：{id(s)}")  # 拼接后：hello world，地址：140708467900272（地址变了）

# 2. 切片替换（看似修改，实际新建）
s = "hello"
s_new = "H" + s[1:]  # 用切片生成新字符串
print(f"替换后：{s_new}，原s地址：{id(s)}，新s_new地址：{id(s_new)}")
# 输出：替换后：Hello，原s地址：140708467898160，新地址：140708467900400（地址不同）
```

#### <2>序列特性（初学者可以先了解，之后再回头看）
字符串是字符的序列，支持「索引」和「切片」操作，可按位置访问或截取字符（与列表、元组的序列操作逻辑一致）
- **索引**：从 0 开始（正向索引），也可从 -1 开始（反向索引，即从末尾倒数）

- **切片**：语法为 **str[start:end:step]**，start 为起始位置（默认 0），end 为结束位置（不包含，默认字符串长度），step 为步长（默认 1，负数表示反向切片）。

#### <3>编码特性
- **UTF-8 编码**：Python 3 中字符串默认采用 UTF-8 编码，支持全球绝大多数字符（中文、英文、符号等），无需额外处理编码问题
- **与字节流转换**：若需与字节流（bytes）转换，可使用 **encode()（字符串转字节流）** 和 **decode()（字节流转字符串）**


### （5）字符串的关键注意事项
- **不可变性避坑**：避免频繁用 + 拼接大量字符串（每次拼接都会生成新对象，效率低），推荐用 str.join() 或列表暂存字符后拼接：
```python
# 低效：频繁 + 拼接
s = ''
for i in range(1000):
    s += str(i)  # 每次循环生成新字符串

# 高效：用列表暂存，最后 join
lst = []
for i in range(1000):
    lst.append(str(i))
s = ''.join(lst)  # 仅生成1个新字符串
```
- **转义字符的使用**：当字符串中需要包含引号、换行、制表符等特殊字符时，可使用转义字符 \：
```python
print('He said: "I\'m fine."')  # He said: "I'm fine."（\转义单引号）
print('第一行\n第二行')  # 换行输出（\n转义为换行）
print('姓名\t年龄')  # 姓名    年龄（\t转义为制表符，空格对齐）
```
- **空字符串与空白字符串的区别**:空字符串（''）长度为 0，无任何字符；空白字符串（'   '）长度为空格个数，包含空白字符：
```python
print(len('') == 0)    # True（空字符串）
print(len('   ') == 0) # False（空白字符串，长度3）
print(''.strip() == '') # True（空字符串strip后仍为空）
print('   '.strip() == '') # True（空白字符串strip后为空）
```
- **字符串比较**：
	- 用 == 比较字符串内容是否相同
	- 用 is 比较是否为同一个对象(不可变特性导致相同内容的短字符串可能复用对象，但不推荐用 is 比较内容）
```python
s1 = 'hello'
s2 = 'hello'
print(s1 == s2)  # True（内容相同）
print(s1 is s2)  # True（短字符串复用对象，地址相同）

s3 = 'hello' * 1000
s4 = 'hello' * 1000
print(s3 == s4)  # True（内容相同）
print(s3 is s4)  # False（长字符串不复用对象，地址不同）
```

# 二、列表（list）
## 1.列表介绍
### （1）列表的定义与基础表示
- **Python 中定义列表使用方括号 [] 包裹元素，元素之间用逗号 , 分隔**
- **列表支持存储不同类型的元素，也支持嵌套（列表内部包含列表）**
- **list()函数**：将**可迭代对象**转换为列表，如字符串、元组
```python
# 1. 基础列表定义（不同类型元素）
lst1 = [10, 3.14, "Python", True]  # 包含 int、float、str、bool 类型
print(lst1)  # [10, 3.14, 'Python', True]

# 2. 空列表定义
lst2 = []  # 空列表，长度为 0
print(len(lst2))  # 0

# 3. 嵌套列表（列表内部包含列表）
lst3 = [1, 2, [3, 4], [5, [6, 7]]]  # 二维、三维嵌套
print(lst3)  # [1, 2, [3, 4], [5, [6, 7]]]

# 4. 用 list() 函数创建列表（可转换可迭代对象，如字符串、元组）
lst4 = list("hello")  # 字符串转列表（每个字符为元素）
lst5 = list((1, 2, 3))  # 元组转列表
print(lst4)  # ['h', 'e', 'l', 'l', 'o']
print(lst5)  # [1, 2, 3]
```

### （2）列表的核心特性
#### <1>可变性（最核心特性）
- **列表创建后，可直接修改元素的值、新增元素、删除元素，无需生成新列表**（与字符串、数字、bool 的不可变性形成对比）。
```python
lst = [1, 2, 3]
# 1. 修改元素（通过索引赋值）
lst[0] = 10  # 将第 0 位元素改为 10
print(lst)  # [10, 2, 3]

# 2. 新增元素（append() 方法）
lst.append(4)  # 在列表末尾新增元素 4
print(lst)  # [10, 2, 3, 4]

# 3. 删除元素（del 语句）
del lst[1]  # 删除第 1 位元素（值为 2）
print(lst)  # [10, 3, 4]
```

#### <2>有序性
- **列表中的元素按「插入顺序」排列，支持通过「索引」（正向、反向）访问元素，也支持切片截取子列表**
	- **正向索引**：从 0 开始，依次递增（0 表示第一个元素，1 表示第二个，以此类推）；
	- **反向索引**：从 -1 开始，依次递减（-1 表示最后一个元素，-2 表示倒数第二个，以此类推）；
	- **切片**：语法为lst[start:end:step]，start 起始位置（默认 0），end 结束位置（不包含，默认列表长度），step 步长（默认 1，负数表示反向切片）。
```python
lst = ["a", "b", "c", "d", "e"]
# 1. 正向索引访问
print(lst[0])  # 'a'（第一个元素）
print(lst[2])  # 'c'（第三个元素）

# 2. 反向索引访问
print(lst[-1])  # 'e'（最后一个元素）
print(lst[-3])  # 'c'（倒数第三个元素）

# 3. 切片操作
print(lst[1:4])  # ['b', 'c', 'd']（从第1位到第3位，不包含第4位）
print(lst[:3])   # ['a', 'b', 'c']（从开头到第2位，不包含第3位）
print(lst[2:])   # ['c', 'd', 'e']（从第2位到末尾）
print(lst[::2])  # ['a', 'c', 'e']（步长2，每隔1个元素取1个）
print(lst[::-1]) # ['e', 'd', 'c', 'b', 'a']（反向切片，倒序输出）
```

#### <3>元素类型无限制
- **列表可存储任意类型的元素，包括数字、字符串、列表、字典等，甚至可以混合存储不同类型。**
```python
lst = [100, "hello", [1,2], {"name": "小明"}, None]
print(lst)  # [100, 'hello', [1, 2], {'name': '小明'}, None]
```

### （3）列表的常用操作
#### <1>元素的增、删、改、查
- **查（访问元素）**：通过索引或切片访问，若索引超出范围会报错（IndexError）；
- **改（修改元素）**：通过索引赋值修改，支持批量修改（切片赋值）；
- **增（新增元素）**：
	- **append()**：末尾新增
	- **insert()**：指定位置新增
	- **extend()**：合并另一个可迭代对象
- **删（删除元素）**：
	- **del**：按索引删除
	- **remove()**：按值删除，删除第一个匹配项
	- **pop()**：按索引删除并返回元素，默认删除末尾
```python
#注：切片是“左闭右开”

lst = [1, 2, 3]
# 1. 查
print(lst[1])  # 2（索引访问）
print(lst[0:2])  # [1, 2]（切片访问）

# 2. 改
lst[2] = 30  # 单个元素修改
print(lst)  # [1, 2, 30]
lst[0:2] = [10, 20]  # 批量修改（切片赋值）
print(lst)  # [10, 20, 30]

# 3. 增
lst.append(40)  # 末尾新增
print(lst)  # [10, 20, 30, 40]
lst.insert(1, 15)  # 在第1位插入 15
print(lst)  # [10, 15, 20, 30, 40]
lst.extend([50, 60])  # 合并列表 [50,60]
print(lst)  # [10, 15, 20, 30, 40, 50, 60]

# 4. 删
del lst[1]  # 按索引删除第1位（15）
print(lst)  # [10, 20, 30, 40, 50, 60]
lst.remove(30)  # 按值删除 30（第一个匹配项）
print(lst)  # [10, 20, 40, 50, 60]
pop_val = lst.pop()  # 默认删除末尾元素，返回删除的值
print(pop_val)  # 60
print(lst)  # [10, 20, 40, 50]
pop_val2 = lst.pop(0)  # 删除第0位元素，返回值
print(pop_val2)  # 10
print(lst)  # [20, 40, 50]
```

#### <2>列表的拼接与重复
- **+**：拼接两个列表
- **\***：将列表元素重复指定次数
```python
lst1 = [1, 2, 3]
lst2 = [4, 5, 6]

# 1. 拼接（+）
lst3 = lst1 + lst2
print(lst3)  # [1, 2, 3, 4, 5, 6]
print(lst1)  # [1, 2, 3]（原列表未修改）

# 2. 重复（*）
lst4 = lst1 * 3
print(lst4)  # [1, 2, 3, 1, 2, 3, 1, 2, 3]
```

#### <3>成员判断（in / not in）
- **用 in 判断元素是否在列表中，not in 判断元素是否不在列表中，返回布尔值**
```python
lst = [10, 20, 30, "hello"]
print(20 in lst)  # True（元素存在）
print(40 in lst)  # False（元素不存在）
print("hello" not in lst)  # False（元素存在，not in 返回 False）
print("world" not in lst)  # True（元素不存在）
```

#### <4>列表的排序与反转
- **sort()**：对列表进行排序，默认升序
- **reverse()**：反转列表
```python
lst = [3, 1, 4, 2, 5]
# 1. sort() 排序（默认升序，修改原列表）
lst.sort()
print(lst)  # [1, 2, 3, 4, 5]
# 降序排序
lst.sort(reverse=True)
print(lst)  # [5, 4, 3, 2, 1]

# 2. sorted() 排序（生成新列表，不修改原列表）
lst2 = [3, 1, 4, 2, 5]
new_lst = sorted(lst2)
print(new_lst)  # [1, 2, 3, 4, 5]
print(lst2)  # [3, 1, 4, 2, 5]（原列表未修改）

# 3. reverse() 反转（修改原列表）
lst.reverse()
print(lst)  # [1, 2, 3, 4, 5]
# 也可用切片反转（生成新列表）
lst3 = lst[::-1]
print(lst3)  # [5, 4, 3, 2, 1]
```

### （4）列表常用内置函数与方法
#### <1>内置函数（全局函数，直接调用）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109232134158.png)

#### <2>列表对象方法（list.方法名()）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109233311166.png)

### （5）列表的关键注意事项
#### <1>可变对象的引用问题
- **列表是可变对象，赋值时传递的是引用（内存地址），修改新列表会影响原列表（与不可变对象的赋值逻辑不同）**
```python
# 可变对象引用传递
lst1 = [1, 2, 3]
lst2 = lst1  # lst2 指向 lst1 的内存地址
lst2[0] = 10  # 修改 lst2，原列表 lst1 也会被修改
print(lst1)  # [10, 2, 3]
print(lst2)  # [10, 2, 3]

# 解决：创建列表副本（不影响原列表）
lst3 = lst1.copy()  # 方法1：copy() 方法
lst4 = lst1[:]      # 方法2：切片（全切片）
lst5 = list(lst1)   # 方法3：list() 函数
lst3[0] = 20
print(lst1)  # [10, 2, 3]（原列表未变）
print(lst3)  # [20, 2, 3]（副本修改）
```

#### <2>嵌套列表的副本问题
- **普通 copy()/切片 是「浅拷贝」，仅复制外层列表，内层嵌套列表仍为引用传递，修改内层列表会影响原列表**
```python
lst1 = [1, 2, [3, 4]]
lst2 = lst1.copy()  # 浅拷贝
lst2[2][0] = 30  # 修改内层列表
print(lst1)  # [1, 2, [30, 4]]（原列表内层被修改）
print(lst2)  # [1, 2, [30, 4]]

# 解决：深拷贝（复制所有层级的元素，完全独立）
import copy
lst3 = copy.deepcopy(lst1)  # 深拷贝
lst3[2][0] = 300
print(lst1)  # [1, 2, [30, 4]]（原列表未变）
print(lst3)  # [1, 2, [300, 4]]
```

#### <3>避免频繁在列表开头增删元素
- **列表是动态数组，开头增删元素（如 insert(0, x)、pop(0)）需要移动所有后续元素，效率低**
- **若需频繁在两端操作，推荐用 collections.deque（双端队列）**
```python
# 低效：列表开头插入
lst = []
for i in range(1000):
    lst.insert(0, i)  # 每次插入都要移动所有元素

# 高效：双端队列开头插入
from collections import deque
dq = deque()
for i in range(1000):
    dq.appendleft(i)  # 双端队列开头插入效率高
```

## 2.操作列表
### （1）列表的创建
#### <1>直接用方括号定义（最基础常用）
- **用 [] 包裹元素，元素间用逗号分隔，支持混合类型元素和嵌套列表**
```python
# 混合类型列表
lst1 = [10, 3.14, "Python", True]
print(lst1)  # 输出：[10, 3.14, 'Python', True]

# 嵌套列表
lst_nest = [1, [2, 3], [4, [5, 6]]]
print(lst_nest)  # 输出：[1, [2, 3], [4, [5, 6]]]
```

#### <2>创建空列表（动态添加元素场景）
- **直接写 []，后续通过 append()、extend() 等方法动态添加元素**
```python
# 创建空列表
lst2 = []
# 动态添加单个元素
lst2.append("test")
print(lst2)  # 输出：['test']

# 继续动态添加多个元素
lst2.extend(["a", "b", "c"])
print(lst2)  # 输出：['test', 'a', 'b', 'c']
```

#### <3>创建数值列表（range()）
##### 1.使用range函数
- **range(start, stop, step)能够轻松地生成一系列的数字（“左闭右开区间”,step：步长）**
```python
for value in range(1,3):
	print(value)
'''
输出：
 1
 2
'''
```

##### 2.使用range()创建数字列表
- **核心语法：list(range(start, stop, step))（start 默认 0，step 默认 1）**
```python
# 语法：list(range(stop))
# 生成：0 ~ stop-1 的整数列表
num_list1 = list(range(6))
print(num_list1)  # 输出：[0, 1, 2, 3, 4, 5]

# 语法：list(range(start, stop))
# 生成：start ~ stop-1 的整数列表（左闭右开）
num_list2 = list(range(3, 9))
print(num_list2)  # 输出：[3, 4, 5, 6, 7, 8]

# 语法：list(range(start, stop, step))
# 生成：从 start 开始，每次加 step，直到不超过 stop（左闭右开）
# 示例1：生成奇数列表（1-9）
odd_nums = list(range(1, 10, 2))
print(odd_nums)  # [1, 3, 5, 7, 9]

# 示例2：生成偶数列表（0-10）
even_nums = list(range(0, 11, 2))
print(even_nums)  # [0, 2, 4, 6, 8, 10]

# 示例3：生成递减数值列表（步长为负）
desc_nums = list(range(10, 0, -1))
print(desc_nums)  # [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
```

#### <4>list()函数转换（可迭代对象转列表）
- **将字符串、元组、range对象等可迭代对象转换为列表**
```python
# 字符串转列表（每个字符为元素）
str_to_lst = list("hello")
print(str_to_lst)  # 输出：['h', 'e', 'l', 'l', 'o']

# 元组转列表
tuple_to_lst = list((1, 2, 3))
print(tuple_to_lst)  # 输出：[1, 2, 3]
```

#### <5>列表推导式（高效生成有规律列表）
- **核心语法：[表达式 for 变量 in 可迭代对象 if 条件]（条件可选）**
```python
# 生成0-9的偶数列表（带条件判断）
even_lst = [i for i in range(10) if i % 2 == 0]
print(even_lst)  # 输出：[0, 2, 4, 6, 8]

# 对已有列表元素加工（无条件判断）
processed_lst = [x*3 for x in [1, 2, 3]]
print(processed_lst)  # 输出：[3, 6, 9]

# 嵌套列表推导式（生成二维列表）
two_dimensional_lst = [[i*j for j in range(3)] for i in range(2)]
print(two_dimensional_lst)  # 输出：[[0, 0, 0], [0, 1, 2]]
```

### （2）列表的遍历（for循环）
#### <1>基础for循环遍历（直接遍历元素）
- **最常用的遍历方式，直接迭代列表中的每个元素，无需关心索引，语法简洁高效，适合仅需使用元素值的场景。**
```python

lst = ["苹果", "香蕉", "橙子"]
# 基础遍历：直接取元素
for fruit in lst:
    print(fruit)

# 遍历混合类型列表
mix_lst = [10, 3.14, "Python"]
for item in mix_lst:
    print(f"元素值：{item}，元素类型：{type(item)}")
# 输出结果：
# 元素值：10，元素类型：<class 'int'>
# 元素值：3.14，元素类型：<class 'float'>
# 元素值：Python，元素类型：<class 'str'>
```

#### <2>带索引的for循环遍历（enumerate函数）
- **当需要同时获取元素的「索引」和「值」时，使用 enumerate() 函数，该函数返回元组 (索引, 元素值)，默认索引从0开始，也可指定起始索引。**
```python

lst = ["A", "B", "C", "D"]
# 默认索引从0开始
for idx, val in enumerate(lst):
    print(f"索引：{idx}，元素：{val}")


# 指定索引从1开始
for idx, val in enumerate(lst, start=1):
    print(f"序号：{idx}，元素：{val}")
```

#### <3>切片遍历（遍历子列表）
**通过切片获取列表的子列表后，再用for循环遍历，适合仅需要遍历列表中部分元素的场景（如前3个元素、间隔元素、倒序元素等）**
```python

lst = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
# 1. 遍历前5个元素（切片：lst[:5]）
print("前5个元素：")
for num in lst[:5]:
    print(num, end=" ")
# 输出：前5个元素：0 1 2 3 4  (注：应该换行，这里为了节省篇幅就不换了)

# 2. 遍历索引2到7的元素（不包含7，切片：lst[2:7]）
print("\n索引2-6的元素：")
for num in lst[2:7]:
    print(num, end=" ")
# 输出：索引2-6的元素：2 3 4 5 6  (注：应该换行，这里为了节省篇幅就不换了)

# 3. 遍历间隔2的元素（步长2，切片：lst[::2]）
print("\n间隔2的元素：")
for num in lst[::2]:
    print(num, end=" ")
# 输出：间隔2的元素：0 2 4 6 8 (注：应该换行，这里为了节省篇幅就不换了)

# 4. 倒序遍历（步长-1，切片：lst[::-1]）
print("\n倒序遍历所有元素：")
for num in lst[::-1]:
    print(num, end=" ")
# 输出：倒序遍历所有元素：9 8 7 6 5 4 3 2 1 0 (注：应该换行，这里为了节省篇幅就不换了)

# 5. 遍历嵌套列表的子列表
nest_lst = [1, 2, [3, 4, 5], 6]
print("\n嵌套列表中的子列表元素：")
for item in nest_lst[2]:  # 先取子列表 lst[2] = [3,4,5]，再遍历
    print(item, end=" ")
# 输出：嵌套列表中的子列表元素：3 4 5 (注：应该换行，这里为了节省篇幅就不换了)
```


# 三、元组
## 1.元组的定义与基础表示
### （1）概述
- **定义元组使用圆括号 () 包裹元素，元素之间用“逗号 , ”分隔**
- **元组支持存储不同类型的元素，也支持嵌套（元组内部包含元组、列表等）**

**注意：当元组中只有一个元素时，必须在元素后加“逗号 , ”，否则圆括号会被当作运算符处理，无法正确创建元组。**

### （2）常用创建方法
```python
# 方法1：直接用圆括号定义（最常用，支持混合类型/嵌套）
t1 = (10, 3.14, "Python", True)  # 包含 int、float、str、bool 类型
t_nest = (1, 2, (3, 4), (5, [6, 7]))  # 二维嵌套元组（内部可包含列表）
print(t1)  # (10, 3.14, 'Python', True)
print(t_nest)  # (1, 2, (3, 4), (5, [6, 7]))

# 方法2：创建单元素元组（必须加逗号，否则不是元组）
t2 = (5,)  # 正确：单元素元组，逗号不可省略
t3 = (5)   # 错误：被当作整数 5 处理，类型是 int
print(type(t2))  # <class 'tuple'>
print(type(t3))  # <class 'int'>

# 方法3：省略圆括号创建（元组的特殊语法，简洁）
t4 = 1, 2, 3  # 无需圆括号，直接用逗号分隔元素
print(t4)  # (1, 2, 3)
print(type(t4))  # <class 'tuple'>

# 方法4：用 tuple() 函数转换（可迭代对象转元组）
t5 = tuple("hello")  # 字符串转元组（每个字符为元素）
t6 = tuple([1, 2, 3])  # 列表转元组
t7 = tuple(range(5))  # range对象转元组（生成连续整数元组）
print(t5)  # ('h', 'e', 'l', 'l', 'o')
print(t6)  # (1, 2, 3)
print(t7)  # (0, 1, 2, 3, 4)

# 方法5：创建空元组
t8 = ()  # 空元组，长度为 0
t9 = tuple()  # 空元组（tuple() 无参数时返回空元组）
print(len(t8))  # 0
print(t9)  # ()
```

## 2.元组的核心特性
### （1）不可变性（最核心特性，必看！）
- **元组存储的是对象的引用，而非对象本身**
- **元组的不可变性仅意味着元组的结构（长度和对象引用）不能改变**
- **但引用指向的可变对象的内部状态可以改变**
```python
t = (1, 2, 3,[3,4])
# 1. 尝试修改元素（报错）
# t[0] = 10  # TypeError: 'tuple' object does not support item assignment
# 2. 尝试删除元素（报错）
# del t[1]  # TypeError: 'tuple' object doesn't support item deletion
# 3. 尝试新增元素（报错）
# t.append(4)  # AttributeError: 'tuple' object has no attribute 'append'

# 4.修改元组中列表的内容（成功）
t[3].append(5)
print(f"修改后元组: {t}")  # 修改后元组: (1, 2, 3，[3, 4, 5])

# 5.尝试替换整个列表（报错）
t[2] = [6, 7, 8]  # 错误：'tuple' object does not support item assignment
```
**注意！！：若元组内部包含“可变对象”（如列表），则可变对象的内部元素可以修改，但元组的结构（长度和对象引用）不能改变**。
- **对于「不可变元素」**（如数字、bool值、字符串等）：引用地址和实际值绑定，修改值 = 修改引用（**由不可变性知**），因此元组不允许操作修改**不可变元素**的值；
	- **不可变元素**：整数（int）、布尔值（bool）、浮点数（float）、字符串（str）、元组（tuple）
- **对于「可变元素」**（如列表、字典等）：引用地址与实际值分开存储，修改值 != 修改引用（**有可变性知**），因此元组允许操作修改**可变元素**内的值；
	- **可变元素**：列表（list）、字典（dict）、集合（set）、字节数组（bytearray）

### （2）有序性
**元组中的元素按「插入顺序」排列，支持通过「索引」（正向、反向）访问元素，也支持切片截取子元组，操作逻辑与列表、字符串完全一致。**
- **正向索引**：从 0 开始，依次递增（0 表示第一个元素）
- **反向索引**：从 -1 开始，依次递减（-1 表示最后一个元素）
- **切片**：语法为 t[start:end:step]，start 起始位置（默认 0），end 结束位置（不包含，默认元组长度），step 步长（默认 1，负数表示反向切片）
```python
t = ("a", "b", "c", "d", "e")
# 1. 索引访问
print(t[0])   # 'a'（正向索引：第0位）
print(t[-1])  # 'e'（反向索引：最后一位）
print(t[3])   # 'd'（正向索引：第3位）

# 2. 切片操作
print(t[1:4])  # ('b', 'c', 'd')（从第1位到第3位，不包含第4位）
print(t[:3])   # ('a', 'b', 'c')（从开头到第2位）
print(t[2:])   # ('c', 'd', 'e')（从第2位到末尾）
print(t[::2])  # ('a', 'c', 'e')（步长2，间隔取元素）
print(t[::-1]) # ('e', 'd', 'c', 'b', 'a')（反向切片，倒序输出）
```

### （3）元素类型无限制
**元组可存储任意类型的元素，包括数字、字符串、元组、列表、字典等，支持混合类型存储。**
```python
t = (100, "hello", (1,2), [3,4], {"name": "小明"}, None)
print(t)  # (100, 'hello', (1, 2), [3, 4], {'name': '小明'}, None)
```

### （4）可哈希性
**由于元组是不可变的，因此它是「可哈希」对象，可以作为字典的键（key），也可以放入集合（set）中；而列表是不可哈希的，无法作为字典键或集合元素。**
```python
# 元组作为字典键（合法）
dict1 = {(1,2): "a", (3,4): "b"}
print(dict1)  # {(1, 2): 'a', (3, 4): 'b'}

# 列表作为字典键（报错）
# dict2 = {[1,2]: "a"}  # TypeError: unhashable type: 'list'

# 元组放入集合（合法）
set1 = {(1,2), (3,4)}
print(set1)  # {(1, 2), (3, 4)}
```

## 3.元组的常见操作
**元组的操作主要围绕「读取」和「判断」展开，因“不可变性”，无增删改相关方法。常用操作包括遍历、索引/切片访问、拼接/重复、成员判断等。**

### （1）元组遍历
- **元组的遍历方式与列表完全一致，支持基础 for 循环、带索引的遍历（enumerate 函数）、切片遍历。**
```python
t = ("苹果", "香蕉", "橙子")

# 1. 基础 for 循环（直接遍历元素）
print("基础遍历：")
for fruit in t:
    print(fruit)  # 苹果、香蕉、橙子

# 2. 带索引的遍历（enumerate 函数）
print("\n带索引遍历：")
for idx, val in enumerate(t, start=1):  # start=1 指定索引从1开始
    print(f"序号：{idx}，元素：{val}")  # 序号1-3，对应元素

# 3. 切片遍历（遍历子元组）
print("\n切片遍历前2个元素：")
for fruit in t[:2]:
    print(fruit)  # 苹果、香蕉

# 4. 嵌套元组遍历
t_nest = (1, 2, (3, 4, 5), 6)
print("\n嵌套元组的子元组遍历：")
for item in t_nest[2]:  # 遍历子元组 t_nest[2] = (3,4,5)
    print(item)  # 3、4、5
```

### （2）拼接与重复
**元组支持用 + 拼接（合并两个元组）、用 \*重复（将元组元素重复指定次数），但这两种操作会生成新元组，不修改原元组（符合不可变特性）。**
```python
t1 = (1, 2, 3)
t2 = (4, 5, 6)

# 1. 拼接（+）
t3 = t1 + t2
print(t3)  # (1, 2, 3, 4, 5, 6)
print(t1)  # (1, 2, 3)（原元组未修改）

# 2. 重复（*）
t4 = t1 * 3
print(t4)  # (1, 2, 3, 1, 2, 3, 1, 2, 3)
```

### （3）成员判断（in / not in）
**用 in 判断元素是否在元组中，not in 判断元素是否不在元组中，返回布尔值。**
```python
t = (10, 20, 30, "hello")
print(20 in t)  # True（元素存在）
print(40 in t)  # False（元素不存在）
print("hello" not in t)  # False（元素存在，not in 返回 False）
print("world" not in t)  # True（元素不存在）
```

## 4.元组的常用内置函数与方法
**元组的方法较少（因不可变性），核心方法为 index()（查找元素索引）和 count()（统计元素出现次数）；常用内置函数与列表通用。**

### （1）内置函数（全局函数，直接调用）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110090007457.png)

### （2）元组对象方法（t.方法名()）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110090252610.png)

## 5.元组与列表的核心区别
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110090555054.png)

## 6.元组的关键注意事项
- **单元素元组必须加逗号**：这是常见的坑！若省略逗号，圆括号会被当作运算符，创建的是对应类型的单个元素，而非元组。
- **元组的不可变是「浅层不可变」**：元组的不可变指的是「元素的引用不可变」，即不能修改元组中元素的指向，但如果元素本身是可变对象（如列表），其内部状态可以改变。
- **元组的「不可变」不代表元素不可变**：元组的不可变指的是「元素的引用不可变」，即不能修改元组中元素的指向，但如果元素本身是可变对象（如列表），其内部状态可以改变。
- **避免不必要的元组转列表**：若仅需读取数据，直接使用元组更高效；若必须修改数据，再转为列表（用 list(tuple)），修改后可再转回元组（用 tuple(list)）。
- **函数返回多个值的本质是元组**：Python 中函数返回多个值时，默认会将这些值封装为元组返回，即使没有显式用圆括号包裹。
```python
# 函数返回多个值（默认元组）
def get_info():
    name = "小明"
    age = 20
    score = 95.5
    return name, age, score  # 等价于 return (name, age, score)

result = get_info()
print(result)  # ('小明', 20, 95.5)
print(type(result))  # <class 'tuple'>

# 解构赋值（直接提取元组元素）
name, age, score = get_info()
print(name)  # 小明
print(age)   # 20
print(score) # 95.5
```