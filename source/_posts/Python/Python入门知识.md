---
title: Python入门知识
date: 2026-01-09 20:22:24
tags:
	- 笔记
categories:
	- Python
---

**阅读建议**：推荐先阅读《Python编程：从入门到实践》作为基础。 
**声明**：本笔记为个人学习 Python 期间整理，部分内容在 AI 辅助下完成。由于水平有限，可能存在错误或表述不当之处，欢迎指正。


# 一、简单类型
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
	- **注意**：不是像某些语言用 && 、 || 、 !

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
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260112103648031.png)

#### <2>字符串对象方法（需通过字符串实例调用，s.方法名()）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109215553043.png)

#### 补充： join()方法
join() 是字符串的内置方法，作用是将一个可迭代对象（列表、元组、字符串、生成器等）中的所有元素拼接成一个新字符串，**拼接的分隔符就是调用 join() 的这个字符串**。
##### 1）基本语法
```python
分隔符字符串.join(可迭代对象)
```
- **要求**：可迭代对象中的元素必须**都是字符串类型**（否则会报错）。
- **返回值**：拼接后的新字符串。

##### 2）基础示例
```python
# 示例1：用逗号拼接列表元素
lst = ["苹果", "香蕉", "橙子"]
result = ",".join(lst)
print(result)  # 输出：苹果,香蕉,橙子

# 示例2：用空格拼接元组元素
tup = ("Python", "Java", "C++")
result = " ".join(tup)
print(result)  # 输出：Python Java C++

# 示例3：空字符串拼接（无分隔符）
chars = ["H", "e", "l", "l", "o"]
result = "".join(chars)
print(result)  # 输出：Hello

# 示例4：复杂分隔符
words = ["2026", "01", "12"]
result = "-".join(words)
print(result)  # 输出：2026-01-12
```
##### 3）常见坑点 & 解决方法
```python
# 坑点 1：可迭代对象含非字符串元素
nums = [1, 2, 3]

# 错误写法（会报错）
try:
    result = "|".join(nums)
except TypeError as e:
    print(f"报错：{e}")
# 输出：报错：sequence item 0: expected str instance, int found

# 正确写法
result = "|".join(str(num) for num in nums)
print(result)  # 输出：1|2|3 ✅

# 坑点 2：拼接空可迭代对象
empty_lst = []
result = "-".join(empty_lst)
print(f"结果：'{result}'")  # 输出：结果：'' 
print(f"长度：{len(result)}")  # 输出：长度：0 

# 坑点 3：误以为是 list.join() 而不是 str.join()
words = ["Hello", "World"]

# ❌ 错误写法（初学者常犯）
# words.join("-")  # 报错：AttributeError: 'list' object has no attribute 'join'

# ✅ 正确写法：是"分隔符".join(列表)
result = "-".join(words)
print(result)  # 输出：Hello-World

```

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

#### <3>编码特性（编码安全）
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

## 补充：decimal 模块（精确浮点运算）
### （1）为什么需要 decimal？
Python 的 float 类型使用二进制浮点数表示，会产生精度丢失：
```python
# 经典的浮点数精度问题
print(0.1 + 0.2)          # 输出：0.30000000000000004（不是 0.3！）
print(0.1 + 0.2 == 0.3)   # 输出：False

# 累计误差问题
total = 0.0
for _ in range(10):
    total += 0.1
print(total)              # 输出：0.9999999999999999（不是 1.0！）

# 金融计算中的灾难
price = 19.99
quantity = 3
print(price * quantity)   # 输出：59.97000000000001（多出 0.00000000000001）
```
**原因**：0.1 在二进制中是无限循环小数，无法精确表示。

### （2）decimal 模块基础用法
#### <1>导入与创建 Decimal 对象
```python
from decimal import Decimal, getcontext

# ⚠️ 关键：用字符串创建，避免 float 的精度问题
d1 = Decimal("0.1")       # ✅ 正确：精确的 0.1
d2 = Decimal("0.2")
d3 = Decimal(0.1)         # ❌ 错误：已经丢失精度
print(d3)                 # 输出：0.1000000000000000055511151231257827021181583404541015625

# 验证精度
print(Decimal("0.1") + Decimal("0.2"))              # 输出：0.3 ✅
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))  # 输出：True ✅
```

#### <2>基本运算
```python
from decimal import Decimal

a = Decimal("10.5")
b = Decimal("3")

print(a + b)    # 输出：13.5
print(a - b)    # 输出：7.5
print(a * b)    # 输出：31.5
print(a / b)    # 输出：3.5
print(a // b)   # 输出：3（整除）
print(a % b)    # 输出：1.5（取余）
print(a ** 2)   # 输出：110.25（幂运算）
```

#### <3>精度控制
##### 1）全局精度控制
```python
from decimal import Decimal, getcontext

# 查看默认精度（28位有效数字）
print(getcontext().prec)  # 输出：28

# 设置全局精度为 6 位有效数字
getcontext().prec = 6

result = Decimal("1") / Decimal("7")
print(result)  # 输出：0.142857（6位有效数字）

# 恢复默认精度
getcontext().prec = 28
```

##### 2）局部精度控制（推荐）
```python
from decimal import Decimal, localcontext

# 使用 localcontext 临时修改精度，不影响全局设置
with localcontext() as ctx:
    ctx.prec = 4
    result = Decimal("1") / Decimal("7")
    print(result)  # 输出：0.1429

# 退出 with 块后恢复原精度
result = Decimal("1") / Decimal("7")
print(result)  # 输出：0.1428571428571428571428571429（28位）
```

#### <4>舍入模式
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260112103214903.png)
```python
from decimal import Decimal, ROUND_HALF_UP, ROUND_DOWN, ROUND_CEILING

price = Decimal("19.996")

# Python 的 ROUND_HALF_UP 是"四舍五入"，但要注意：
# 0.5 会向远离零的方向舍入（即正数向上，负数向下）
# 四舍五入到 2 位小数（常用于金融）
print(price.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP))  # 输出：20.00

# 直接截断（向零取整）
print(price.quantize(Decimal("0.01"), rounding=ROUND_DOWN))     # 输出：19.99

# 向上取整（正数向正无穷，负数向零）
print(price.quantize(Decimal("0.01"), rounding=ROUND_CEILING))  # 输出：20.00

# 负数的向上取整
neg_price = Decimal("-19.996")
print(neg_price.quantize(Decimal("0.01"), rounding=ROUND_CEILING))  # 输出：-19.99
```

#### <5>Decimal 与其他类型的转换
```python
from decimal import Decimal

# Decimal → float（可能丢失精度）
d = Decimal("3.14159265358979323846")
f = float(d)
print(f)  # 输出：3.141592653589793（精度丢失）

# Decimal → int（截断小数部分）
d = Decimal("99.99")
i = int(d)
print(i)  # 输出：99

# Decimal → str
d = Decimal("123.45")
s = str(d)
print(s)  # 输出：123.45

# float → Decimal（⚠️ 不推荐，精度已丢失）
d = Decimal(0.1)  # 不要这样做！

# float → Decimal（正确做法：先转字符串）
d = Decimal(str(0.1))  # ✅
```

### （3）性能对比与使用建议
```python
import timeit
from decimal import Decimal

# 性能对比
float_time = timeit.timeit("0.1 + 0.2", number=1000000)
decimal_time = timeit.timeit(
    "Decimal('0.1') + Decimal('0.2')", 
    setup="from decimal import Decimal",
    number=1000000
)

print(f"float 运算时间：{float_time:.4f}s")
print(f"Decimal 运算时间：{decimal_time:.4f}s")
# Decimal 比 float 慢约 10-100 倍
```
**使用建议**：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260112000424733.png)

### （4）常见陷阱与避坑
```python
from decimal import Decimal

# ❌ 陷阱1：用 float 创建 Decimal
bad = Decimal(0.1)
print(bad)  # 0.1000000000000000055511151231257827...

# ✅ 正确：用字符串创建
good = Decimal("0.1")
print(good)  # 0.1

# ❌ 陷阱2：Decimal 与 float 混合运算
d = Decimal("10.5")
f = 3.0
# result = d + f  # TypeError: unsupported operand type(s)

# ✅ 正确：统一类型
result = d + Decimal(str(f))
print(result)  # 13.5

# ❌ 陷阱3：忘记处理精度
price = Decimal("19.99") * Decimal("0.1")  # 打9折
print(price)  # 1.999（可能需要四舍五入）

# ✅ 正确：明确精度
price = (Decimal("19.99") * Decimal("0.1")).quantize(Decimal("0.01"))
print(price)  # 2.00
```

---
# 二、列表（list）
## 1.列表的定义与基础表示
### （1）概述
- **Python 中定义列表使用方括号 [] 包裹元素，元素之间用逗号 , 分隔**
- **列表支持存储不同类型的元素，也支持嵌套（列表内部包含列表）**
- **list()函数**：将**可迭代对象**转换为列表，如字符串、元组

### （2）常用创建方法
```python
# 方法1： 基础列表定义（不同类型元素）
lst1 = [10, 3.14, "Python", True]  # 包含 int、float、str、bool 类型
print(lst1)  # [10, 3.14, 'Python', True]

# 方法2：空列表定义
lst2 = []  # 空列表，长度为 0
print(len(lst2))  # 0

# 方法3：嵌套列表（列表内部包含列表）
lst3 = [1, 2, [3, 4], [5, [6, 7]]]  # 二维、三维嵌套
print(lst3)  # [1, 2, [3, 4], [5, [6, 7]]]

# 方法4：用 list() 函数创建列表（可转换可迭代对象，如字符串、元组）
lst4 = list("hello")  # 字符串转列表（每个字符为元素）
lst5 = list((1, 2, 3))  # 元组转列表
print(lst4)  # ['h', 'e', 'l', 'l', 'o']
print(lst5)  # [1, 2, 3]
```

## 2.列表的核心特性
### （1）可变性（最核心特性）
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

### （2）有序性
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

### （3）元素类型无限制
- **列表可存储任意类型的元素，包括数字、字符串、列表、字典等，甚至可以混合存储不同类型。**
```python
lst = [100, "hello", [1,2], {"name": "小明"}, None]
print(lst)  # [100, 'hello', [1, 2], {'name': '小明'}, None]
```

### （4）不可哈希性
**由于列表是可变对象，因此它是「不可哈希」的，无法作为字典的键（key），也无法放入集合（set）中（集合元素需可哈希）。**
```python
# 列表作为字典键（报错）
# dict1 = {[1,2]: "error"}  # TypeError: unhashable type: 'list'

# 列表放入集合（报错）
# set1 = {[1,2], [3,4]}  # TypeError: unhashable type: 'list'
```

## 3.列表的常用操作
### （1）元素的增、删、改、查
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

**补充：切片赋值的长度可以不一致**
```python
lst = [1, 2, 3, 4, 5]
lst[1:3] = [20, 30, 40, 50]  # 用4个元素替换2个元素
print(lst)  # [1, 20, 30, 40, 50, 4, 5]

lst[1:5] = [100]  # 用1个元素替换4个元素
print(lst)  # [1, 100, 4, 5]
```


### （2）列表的拼接与重复
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

### （3）成员判断（in / not in）
- **用 in 判断元素是否在列表中，not in 判断元素是否不在列表中，返回布尔值**
```python
lst = [10, 20, 30, "hello"]
print(20 in lst)  # True（元素存在）
print(40 in lst)  # False（元素不存在）
print("hello" not in lst)  # False（元素存在，not in 返回 False）
print("world" not in lst)  # True（元素不存在）
```

### （4）列表的排序与反转
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

# 2. reverse() 反转（修改原列表）
# 当前 lst 为 [5, 4, 3, 2, 1]，反转后变为 [1, 2, 3, 4, 5]
lst.reverse()
print(lst)  # [1, 2, 3, 4, 5]

```

## 4.列表常用内置函数与方法
### （1）内置函数（全局函数，直接调用）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109232134158.png)

### （2）列表对象方法（list.方法名()）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109233311166.png)



## 5.操作列表
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

## 6.列表的遍历（for循环）
### （1）基础for循环遍历（直接遍历元素）
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

### （2）带索引的for循环遍历（enumerate函数）
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

### （3）切片遍历（遍历子列表）
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

## 7.列表的关键注意事项
### （1）可变对象的引用问题（“可变性”）
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

### （2）嵌套列表的副本问题
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

### （3）避免频繁在列表开头增删元素
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

---
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
print(f"修改后元组: {t}")  # 修改后元组: (1, 2, 3,[3, 4, 5])

# 5.尝试替换整个列表（报错）
t[3] = [6, 7, 8]  # 错误：'tuple' object does not support item assignment
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

---
# 四、字典
## 1.字典的定义与基础表示
### （1）概述
- **定义字典使用花括号 {} 包裹键值对，键与值之间用冒号 : 分隔，不同键值对之间用逗号 , 分隔**
	- **键（key）**：必须是**不可变对象**（如整数、字符串、元组），且在同一个字典中**唯一**，**不可重复**；
	- **值（value）**：可以是任意类型的对象（如整数、字符串、列表、字典等），**支持重复**；
	- **无序性**：Python 3.7 之前字典是无序的，Python 3.7 及以后保证插入顺序，但仍不支持索引访问（需通过键访问）

### （2）常用创建方法
```python
# 方法1：直接用花括号定义（最常用，支持混合类型值）
dict1 = {"name": "小明", "age": 20, "score": 95.5, "is_pass": True}
# 键：字符串类型；值：字符串、整数、浮点数、布尔值
print(dict1)  # {'name': '小明', 'age': 20, 'score': 95.5, 'is_pass': True}

# 方法2：创建空字典（动态添加键值对场景）
dict2 = {}  # 空字典，长度为 0
dict3 = dict()  # 空字典（dict() 无参数时返回空字典）
print(len(dict2))  # 0
print(dict3)  # {}

# 方法3：用 dict() 函数转换（多种可迭代对象转字典）
# 3.1 列表/元组嵌套键值对
dict4 = dict([("name", "小红"), ("age", 19)])
dict5 = dict((("name", "小刚"), ("age", 21)))
print(dict4)  # {'name': '小红', 'age': 19}
print(dict5)  # {'name': '小刚', 'age': 21}

# 3.2 关键字参数（键为字符串时可用）
dict6 = dict(name="小李", age=22, score=88.0)
print(dict6)  # {'name': '小李', 'age': 22, 'score': 88.0}

# 3.3 两个可迭代对象（键列表+值列表，用 zip 打包）
keys = ["name", "age"]
values = ["小王", 23]
dict7 = dict(zip(keys, values))
print(dict7)  # {'name': '小王', 'age': 23}

# 方法4：字典推导式（简洁生成有规律的字典，高效）
dict8 = {i: i*2 for i in range(5)}  # 键为 0-4，值为键的 2 倍
dict9 = {k: v.upper() for k, v in {"a": "apple", "b": "banana"}.items()}
print(dict8)  # {0: 0, 1: 2, 2: 4, 3: 6, 4: 8}
print(dict9)  # {'a': 'APPLE', 'b': 'BANANA'}
```
**补充**：
- **zip(\*iterables)**：将多个可迭代对象（如列表、元组等）对应位置的元素打包成一个个元组，返回一个可迭代的 zip 对象。
- **字典推导式语法**：{key_expr: value_expr for item in iterable if condition}


## 2.字典的核心特性
### （1）键值对映射（核心特性）
- **字典的核心作用是通过「键」快速定位「值」，无需像列表/元组那样遍历查找，访问效率极高（时间复杂度 O(1)）**
- **键是查找的唯一标识，必须满足「不可变」和「唯一」两个条件。**
```python
dict1 = {"name": "小明", "age": 20}
# 通过键访问值（核心用法）
print(dict1["name"])  # 小明
print(dict1["age"])   # 20

# 键必须唯一，重复键会覆盖前面的值
dict2 = {"a": 1, "a": 2}
print(dict2)  # {'a': 2}（后面的键值对覆盖前面的）

# 键必须是不可变对象，用可变对象作键会报错
# dict3 = {[1,2]: "error"}  # TypeError: unhashable type: 'list'
# 合法的键类型：整数、字符串、元组
dict4 = {1: "int", "str": 2, (3,4): 3}
print(dict4)  # {1: 'int', 'str': 2, (3, 4): 3}
```

### （2）可变性（支持动态增删改）
**字典创建后，可自由添加新的键值对、修改已有键对应的值、删除不需要的键值对，且修改后字典对象的内存地址不变（直接修改原对象）。**
```python
dict1 = {"name": "小明", "age": 20}
print(f"修改前地址：{id(dict1)}")  # 如：140692251235840

# 1. 修改已有键的值
dict1["age"] = 21
print(dict1)  # {'name': '小明', 'age': 21}

# 2. 添加新键值对（键不存在则新增）
dict1["score"] = 95.5
print(dict1)  # {'name': '小明', 'age': 21, 'score': 95.5}

# 3. 删除键值对
del dict1["score"]
print(dict1)  # {'name': '小明', 'age': 21}

print(f"修改后地址：{id(dict1)}")  # 如：140692251235840（地址不变）
```

### （3）无序性（Python 3.7+ 有条件有序）
- **python 3.6 之前**：字典完全无序，每次迭代可能顺序不同
- **Python 3.7+**：正式将保留插入顺序作为语言特性
- **无论是否有序，字典的核心访问方式都是通过键，而非位置。**
- **保留插入顺序不等于排序**：字典不会自动对键进行排序
```python
dict1 = {"b": 2, "a": 1, "c": 3}
print(dict1)  # Python 3.7+ 输出：{'b': 2, 'a': 1, 'c': 3}（保留插入顺序）

# 不支持索引访问，报错
# print(dict1[0])  # TypeError: 'dict' object is not subscriptable
```

### （4）值可任意性与可重复性
**字典的值可以是任意类型的对象（包括可变对象如列表、字典），且多个键可以对应相同的值（值不唯一）。**
```python
dict1 = {
    "list": [1,2,3],  # 值为列表（可变）
    "dict": {"a": 1},  # 值为字典（可变）
    "tuple": (4,5),   # 值为元组（不可变）
    "repeat_val": 10  # 重复值
}
dict1["another_repeat"] = 10  # 新增键，值与上面重复
print(dict1)
# 输出：{'list': [1, 2, 3], 'dict': {'a': 1}, 'tuple': (4, 5), 'repeat_val': 10, 'another_repeat': 10}
```
### （5）不可哈希性
**由于字典是可变对象，因此它是「不可哈希」的，无法作为字典的键（key），也无法放入集合（set）中（集合元素需可哈希）。**
```python
# 字典作为字典键（报错）
# dict1 = {{"a":1}: "error"}  # TypeError: unhashable type: 'dict'

# 字典放入集合（报错）
# set1 = {{"a":1}, {"b":2}}  # TypeError: unhashable type: 'dict'
```

## 3.字典的常用操作
**字典的操作围绕「键值对的增、删、改、查」展开，提供了丰富的内置方法，核心操作包括访问值、添加/修改键值对、删除键值对、遍历字典等。**

### （1）键值对“增加操作”（含“更新操作”）
**字典的增加操作核心是向字典中添加新的键值对，常用方式有3种，根据场景灵活选择：**
- **直接赋值（最常用）**：通过 字典[新键] = 新值 的形式添加，**若键不存在则新增，若键已存在则覆盖原有值**；
- **setdefault() 方法**：添加键值对，**若键已存在则返回原有值（不覆盖），若键不存在则添加并返回默认值（如果不设置默认值，则返回None）**；
- **update() 方法**：批量添加键值对，支持**传入字典**或**键值对参数**,**若键不存在则新增，若键已存在则覆盖原有值**
```python
# 1. 直接赋值（新增键值对）
dict1 = {"name": "小明", "age": 20}
dict1["score"] = 95.5  # 键"score"不存在，新增
dict1["gender"] = "男"  # 键"gender"不存在，新增
print(dict1)  # {'name': '小明', 'age': 20, 'score': 95.5, 'gender': '男'}

# 注意：键已存在时会覆盖
dict1["age"] = 21  # 键"age"已存在，覆盖原有值
print(dict1)  # {'name': '小明', 'age': 21, 'score': 95.5, 'gender': '男'}

# 2. setdefault() 方法（新增不覆盖）
dict2 = {"name": "小红", "age": 19}
# 键不存在，新增并返回默认值
hobby = dict2.setdefault("hobby", "画画")
print(hobby)  # 画画
print(dict2)  # {'name': '小红', 'age': 19, 'hobby': '画画'}

# 键不存在，且不设置默认值，返回None
d = {}
result = d.setdefault("key")
print(result)  # None
print(d)       # {'key': None}


# 键已存在，返回原有值，不覆盖
age = dict2.setdefault("age", 20)
print(age)  # 19
print(dict2)  # {'name': '小红', 'age': 19, 'hobby': '画画'}

# 3. update() 方法（批量新增）
dict3 = {"name": "小刚", "age": 22}
# 传入字典批量新增
dict3.update({"score": 88.0, "address": "北京"})
print(dict3)  # {'name': '小刚', 'age': 22, 'score': 88.0, 'address': '北京'}

# 传入键值对参数批量新增
dict3.update(gender="男", hobby="篮球")
print(dict3)  # {'name': '小刚', 'age': 22, 'score': 88.0, 'address': '北京', 'gender': '男', 'hobby': '篮球'}
```

### （2）访问字典的值（3种常用方式）
- **方括号 [] 访问**：直接通过键访问，键不存在则报错（KeyError）
- **get() 方法访问**：键不存在时返回默认值（默认 None，可手动指定），不会报错（**推荐使用**）
- **values() 方法**：获取所有值组成的可迭代对象（视图对象），可用于转为列表等
```python
dict1 = {"name": "小明", "age": 20, "score": 95.5}

# 1. 方括号 [] 访问
print(dict1["name"])  # 小明
# print(dict1["gender"])  # KeyError: 'gender'（键不存在报错）

# 2. get() 方法访问（推荐）
print(dict1.get("age"))  # 20
print(dict1.get("gender"))  # None（键不存在，返回默认值）
print(dict1.get("gender", "未知"))  # 未知（指定默认值）

# 3. values() 方法（获取所有值）
values = dict1.values()
print(values)  # dict_values(['小明', 20, 95.5])（视图对象）
print(list(values))  # ['小明', 20, 95.5]（转为列表）
```

### （3）“获取、判断”字典的键
- **keys() 方法**：获取所有键组成的可迭代对象（视图对象）,可用于转为列表等
- **items() 方法**：获取所有键值对组成的可迭代对象（**每个元素是 (key, value) 元组**）
- **in / not in**：判断键是否存在于字典中（**判断的是键，不是值**）
```python
dict1 = {"name": "小明", "age": 20, "score": 95.5}

# 1. keys() 方法（获取所有键）
keys = dict1.keys()
print(keys)  # dict_keys(['name', 'age', 'score'])（视图对象）
print(list(keys))  # ['name', 'age', 'score']（转为列表）

# 2. items() 方法（获取所有键值对）
items = dict1.items()
print(items)  # dict_items([('name', '小明'), ('age', 20)])（视图对象）
print(list(items))  # [('name', '小明'), ('age', 20)]（转为列表）

# 3. 成员判断（in / not in）
print("name" in dict1)  # True（键存在）
print("gender" in dict1)  # False（键不存在）
print("小明" in dict1)  # False（判断的是键，不是值）
```

### （4）删除键值对（4种常用方式）
- **del 语句**：通过键删除指定键值对，键不存在则报错
- **pop() 方法**：删除指定键的键值对，并返回对应的值，键不存在可指定默认值
- **popitem() 方法**：删除并返回最后插入的键值对（元组形式），空字典调用报错
- **clear() 方法**：清空字典中所有键值对，返回空字典
```python
dict1 = {"name": "小明", "age": 20, "score": 95.5, "gender": "男"}

# 1. del 语句
del dict1["gender"]
print(dict1)  # {'name': '小明', 'age': 20, 'score': 95.5}
# del dict1["address"]  # KeyError: 'address'（键不存在报错）

# 2. pop() 方法
score = dict1.pop("score")
print(score)  # 95.5（返回删除的值）
print(dict1)  # {'name': '小明', 'age': 20}
# 键不存在时指定默认值
address = dict1.pop("address", "未知")
print(address)  # 未知

# 3. popitem() 方法（删除最后插入的键值对）
item = dict1.popitem()
print(item)  # ('age', 20)（返回删除的键值对元组）
print(dict1)  # {'name': '小明'}

# 4. clear() 方法（清空字典）
dict1.clear()
print(dict1)  # {}
```

### （5）字典的遍历（3种核心场景）
**核心逻辑：字典默认遍历的是键，通过 keys()、values()、items() 方法可分别获取“键”、“值”、“键值对”的可迭代对象（视图对象），进而实现针对性遍历。**
```python
dict1 = {"name": "小明", "age": 20, "score": 95.5}

# 1. 遍历键（最常用，默认遍历键）
print("遍历键：")
for key in dict1:
    print(key)  # name、age、score
# 等价于：for key in dict1.keys():

# 2. 遍历值
print("\n遍历值：")
for value in dict1.values():
    print(value)  # 小明、20、95.5

# 3. 遍历键值对（最实用，通过 items() 方法）
print("\n遍历键值对：")
for key, value in dict1.items():
    print(f"{key}: {value}")
# 输出：
# name: 小明
# age: 20
# score: 95.5
```

### 补充：视图对象
#### <1>视图对象的核心定义
**视图对象是通过字典的 keys()、values()、items() 方法返回的对象（分别对应「键视图」「值视图」「键值对视图」），特点是：**
- **动态关联**：视图对象**不存储数据**，而是实时映射原字典的内容 —— **原字典修改后，视图对象会立即反映出变化**；
- **不可修改**：视图对象本身不能直接增删改（比如不能给 dict.keys() 返回的对象加元素），但可通过原字典修改，视图会同步；
- **可迭代**：可以用 for 循环遍历，也支持部分集合操作（如判断元素是否存在）。

#### <2>三类视图对象及基础用法
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110102612554.png)
```python
# 初始化字典
student = {"name": "张三", "age": 25, "city": "北京"}

# 1. 键视图（dict_keys）
key_view = student.keys()
print("键视图对象：", key_view)  # 键视图对象： dict_keys(['name', 'age', 'city'])
print("遍历键视图：", end=" ")
for k in key_view:
    print(k, end=" ")  # 遍历键视图： name age city
print()

# 2. 值视图（dict_values）
value_view = student.values()
print("值视图对象：", value_view)  # 值视图对象： dict_values(['张三', 25, '北京'])

# 3. 键值对视图（dict_items）
item_view = student.items()
print("键值对视图对象：", item_view)  # 键值对视图对象： dict_items([('name', '张三'), ('age', 25), ('city', '北京')])
for k, v in item_view:
    print(f"{k}: {v}")  # 逐行输出 name: 张三 / age:25 / city:北京
```

#### <3>视图对象的核心特性（重点）
##### 特性1：动态关联原字典（最核心）
**视图对象不复制数据，原字典修改后，视图会实时更新，这是和列表最大的区别**
```python
# 初始化字典
student = {"name": "张三", "age": 25, "city": "北京"}

# 键视图（dict_keys）
key_view = student.keys()
# 值视图（dict_values）
value_view = student.values()


print("修改前值视图：", value_view)  # 修改前值视图： dict_values(['张三', 25, '北京'])

# 修改原字典的内容
student["age"] = 26  # 覆盖已有值
student["gender"] = "男"  # 新增键值对

# 视图对象自动同步变化（无需重新获取）
print("修改后值视图：", value_view)  # 修改后值视图： dict_values(['张三', 26, '北京', '男'])
print("修改后键视图：", key_view)    # 修改后键视图： dict_keys(['name', 'age', 'city', 'gender'])
```
##### 特性2：不可直接修改，但可判断元素是否存在
**图对象本身不能增删改，但若想修改内容，需操作原字典；同时支持 in 操作判断元素是否存在**
```python
# 初始化字典
student = {"name": "张三", "age": 25, "city": "北京"}

# 键视图（dict_keys）
key_view = student.keys()
# 值视图（dict_values）
value_view = student.values()
# 键值对视图（dict_items）
item_view = student.items()
# 1. 判断元素是否存在（支持）
print("name" in key_view)  # True（键视图判断键是否存在）
print(25 in value_view)    # True（值视图判断值是否存在）
print(('age', 25) in item_view)  # True（键值对视图判断键值对是否存在）

# 2. 尝试直接修改视图（报错）
try:
    key_view.append("hobby")  # 视图对象无append方法
except AttributeError as e:
    print("报错：", e)  # 报错： 'dict_keys' object has no attribute 'append'

# 3. 正确修改方式：操作原字典，视图自动同步
student["hobby"] = "篮球"
print("新增hobby后键视图：", key_view)  # 修改后键视图： dict_keys(['name', 'age', 'city', 'gender','hobby'])
```

##### 特性3：可转为列表/集合（按需复制数据）
**若需要对“键/值/键值对”做增删改、切片等操作，可将视图对象转为列表/集合（此时会复制数据，不再关联原字典）：**
```python
# 视图转列表（复制数据，后续修改列表不影响原字典）
key_list = list(key_view)
key_list.append("score")
print("列表新增元素：", key_list)  # ['name', 'age', 'city', 'gender', 'hobby', 'score']
print("原字典键视图：", key_view)  # 仍为 ['name', 'age', 'city', 'gender', 'hobby']（无score）

# 视图转集合（支持集合运算）
value_set = set(value_view)
print("值集合：", value_set)  # {'张三', 26, '北京', '男', '篮球'}
```

## 4.字典的常用内置函数与方法
### （1）内置函数（全局函数，直接调用）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110103442591.png)

### （2）字典对象方法（d.方法名()）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110103756223.png)

## 5.字典与元组、列表的核心区别
**字典、元组、列表是 Python 中最常用的三种容器类型，核心差异在于「数据组织方式」和「可变性」，适用场景完全不同。以下是详细对比：**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110104027376.png)

## 6.字典的关键注意事项
- **键的选择原则**
	- 优先使用字符串作为键（语义清晰，通用性强）
	- 避免使用复杂不可变对象（如嵌套元组）
	- **键必须唯一，重复键会覆盖前面的值**
- **访问值的推荐方式**
	- 优先使用 get() 方法访问值，而非方括号 []，可避免键不存在时的 KeyError 报错，提升代码稳定性。
- **字典的浅拷贝与深拷贝**
	- **浅拷贝**（d.copy()、dict(d)）：仅复制字典本身，值为引用（若值是可变对象，修改原字典的值会影响拷贝后的字典）；
	- **深拷贝**（需导入 copy 模块，copy.deepcopy(d)）：完全复制字典及所有值对象，原字典与拷贝字典相互独立。
- **避免在遍历字典时修改结构**
	- 遍历字典（如 for key in d）时，**禁止直接添加/删除键值对（会导致遍历异常）**
	- 若需修改，可先遍历字典的键列表（for key in list(d.keys())）
- **空字典的判断**
	- 判断字典是否为空，**优先使用 if not d:（简洁高效）**，而非 if len(d) == 0:

```python
# 1. 浅拷贝 vs 深拷贝示例
import copy
d1 = {"name": "小明", "hobbies": ["篮球", "游戏"]}
d2 = d1.copy()  # 浅拷贝
d3 = copy.deepcopy(d1)  # 深拷贝

# 修改原字典的可变值（hobbies列表）
d1["hobbies"].append("读书")
print(d2["hobbies"])  # ['篮球', '游戏', '读书']（浅拷贝受影响）
print(d3["hobbies"])  # ['篮球', '游戏']（深拷贝不受影响）

# 2. 遍历字典时修改结构（正确做法）
d4 = {"a":1, "b":2, "c":3}
# 错误：直接遍历字典时删除键
# for key in d4:
#     del d4[key]  # RuntimeError: dictionary changed size during iteration

# 正确：遍历键列表
for key in list(d4.keys()):
    del d4[key]
print(d4)  # {}（正常清空）

# 3. 空字典判断
d5 = {}
if not d5:
    print("字典为空")  # 输出：字典为空
```

---
# 五、分支结构（if、match-case）
## 1.if 语句：灵活适配复杂条件的基础分支 
- **if 语句是 Python 中最基础、最常用的分支结构，核心逻辑是「判断条件表达式的布尔值（True/False），执行对应代码块」。**
- **支持单分支、双分支、多分支三种形式，可灵活应对从简单到复杂的各类条件判断场景。**

### （1）核心语法与结构（有基础的可跳过这节）
**Python 中 if 语句的语法核心是「条件表达式 + 缩进代码块」，缩进（通常 4 个空格）是区分代码块归属的关键（无大括号 {} 包裹）。**

#### <1>单分支：满足条件才执行
**适用场景：仅需在某个条件成立时执行一段代码，不成立则跳过**
```python
# 语法结构
if 条件表达式:
    缩进的代码块  # 条件为 True 时执行，False 则跳过

# 实战示例：判断数值是否为正数
num = 15
if num > 0:
    print(f"{num} 是正数")  # 输出：15 是正数

# 条件不成立时的情况
num = -3
if num > 0:
    print(f"{num} 是正数")  # 无输出（条件为 False，代码块跳过）
```

#### <2>双分支：非此即彼的二选一
**适用场景：存在两种互斥条件，无论哪种情况都需执行对应代码（必走一个分支）**
```python
# 语法结构
if 条件表达式:
    代码块1  # 条件为 True 时执行
else:
    代码块2  # 条件为 False 时执行

# 实战示例：判断成绩是否合格
score = 75
if score >= 60:
    print("成绩合格，顺利通过")  # 输出：成绩合格，顺利通过
else:
    print("成绩不合格，需补考")
```

#### <3>三元表达式：双分支的简洁写法
- **三元表达式是 if-else 双分支的简化形式，适用于两个分支均只有单条语句、且目的是返回一个值的场景**
- **它能让代码更简洁紧凑，但复杂逻辑下不建议使用（会降低可读性）**
```python
# 核心语法
结果变量 = 表达式1 if 条件表达式 else 表达式2
# 逻辑：条件表达式为 True 时，结果变量 = 表达式1；否则 = 表达式2

# 基础示例：判断成绩合格与否
score = 75
result = "合格" if score >= 60 else "不合格"
print(f"成绩等级：{result}")  # 输出：成绩等级：合格

# 对比普通 if-else（功能等价，三元表达式更简洁）
score = 75
if score >= 60:
    result = "合格"
else:
    result = "不合格"
print(f"成绩等级：{result}")  # 输出：成绩等级：合格

# 进阶示例：用于简单计算
a = 10
b = 20
max_num = a if a > b else b
print(f"较大值：{max_num}")  # 输出：较大值：20

# 注意：三元表达式支持嵌套，但不推荐（可读性差）
# 嵌套示例：判断分数等级（及格/良好/优秀简化版）
score = 88
level = "优秀" if score >= 90 else ("良好" if score >= 80 else "及格")
print(f"成绩等级：{level}")  # 输出：成绩等级：良好
```

#### <4>多分支：按顺序匹配多个条件
**适用场景：存在多个递进条件，按顺序判断，匹配到第一个成立的条件后执行对应代码，后续条件不再判断（避免重复执行）。**
```python
# 语法结构
if 条件表达式1:
    代码块1  # 条件1为 True 时执行
elif 条件表达式2:  # 可添加多个 elif（else if 的缩写）
    代码块2  # 条件2为 True 时执行
elif 条件表达式3:
    代码块3  # 条件3为 True 时执行
...
else:  # 可选，所有条件都不成立时执行
    代码块n

# 实战示例：根据分数划分等级
score = 88
if score >= 90:
    print("等级：优秀")
elif score >= 80:
    print("等级：良好")  # 输出：等级：良好（第一个匹配的条件）
elif score >= 60:
    print("等级：及格")
else:
    print("等级：不及格")

# 注意：多分支的条件顺序不可随意调换
# 错误示例：若先判断 score >=60，会导致 88 直接匹配到及格，后续条件失效
score = 88
if score >= 60:
    print("等级：及格")  # 错误输出：等级：及格
elif score >= 80:
    print("等级：良好")
elif score >= 90:
    print("等级：优秀")
```

### （2）条件表达式的合法形式
**if 语句的条件表达式最终需返回布尔值（True/False），常见合法形式包括：**
- **直接布尔值**：True/False 直接作为条件
- **比较运算**：==（等于）、!=（不等于）、>（大于）、<（小于）、>=（大于等于）、<=（小于等于）
- **逻辑运算**
	- **and（与）**：所有条件都成立才为 True
	- **or （或）**：任意一个条件成立即为 True
	- **not（非）**：取反）
- **成员判断**
	- **in**：元素在容器中
	- **not in**：元素不在容器中
- **身份判断**
	- **is**：两个对象是同一内存地址
	- **is not**：两个对象不是同一内存地址
- **对象（自动转为bool值）**
	- **数字**：0 为 False，非 0 为 True
	- **字符串**：空字符串为 False，非空为 True
	- **容器**：空列表/字典/元组为 False，非空为 True

```python
# 1. 比较运算 + 逻辑运算
age = 22
if age >= 18 and age <= 30:
    print("青年")  # 输出：青年

# 2. 成员判断
fruits = ["apple", "banana", "orange"]
if "banana" in fruits:
    print("包含香蕉")  # 输出：包含香蕉

# 3. 身份判断（判断是否为 None）
obj = None
if obj is None:
    print("对象为空")  # 输出：对象为空

# 4. 自动转换为布尔值的对象
s = "hello"
if s:  # 非空字符串 → True
    print("字符串非空")  # 输出：字符串非空

lst = []
if not lst:  # 空列表 → False，not 取反为 True
    print("列表为空")  # 输出：列表为空
```

### （3）if 语句的避坑要点
- **缩进错误是致命问题**
	- 代码块必须缩进（4 个空格），且同一级代码块缩进一致；
	- 若忘记缩进或缩进混乱，会报 IndentationError 错误；
- **不要混淆 == 和 is**
	- **==**： 判断值是否相等
	- **is**：判断内存地址是否相同
	- 通常判断值用 ==，判断是否为 None 用 is
- **多分支条件需按「从严格到宽松」排序**
	- 如划分成绩等级时，需先判断高分段（90+），再判断低分段（80+），否则会出现逻辑错误
- **避免冗余条件**：
	- 多分支中，后续条件默认是「前面条件不成立」的前提下判断，无需重复添加前面的条件（如无需写 if score >=80 and score <90）

## 2.case分支（Python 3.10+ 新增的模式匹配）
### （1）核心语法与结构
**match-case 的语法核心是「匹配表达式 + 多个 case 模式 + 代码块」，每个 case 对应一种匹配规则，最后可通过通配符 _ 匹配所有未命中的情况。**
```python
# 基础语法结构
match 匹配表达式:  # 需匹配的值/表达式（如变量、函数返回值）
    case 模式1:  # 第一个匹配模式
        代码块1  # 匹配成功则执行，执行后自动终止（无需 break）
    case 模式2:  # 第二个匹配模式
        代码块2  # 模式1不匹配时，判断模式2
    case 模式3 | 模式4:  # 多模式匹配（用 | 分隔，满足一个即可）
        代码块3
    case _:  # 通配符，匹配所有未匹配的情况（类似 else，可选）
        代码块n

# 最简单示例：精准匹配值
day = 3
match day:
    case 1:
        print("星期一")
    case 2:
        print("星期二")
    case 3:
        print("星期三")  # 输出：星期三
    case 4 | 5:  # 多模式匹配：4 或 5
        print("工作日")
    case 6 | 7:
        print("周末")
    case _:
        print("无效的日期")
```

### （2）常用匹配模式（核心重点）
**match-case 的强大之处在于支持多种灵活的模式，而非仅能匹配固定值，以下是最常用的 6 种模式：**
#### <1>字面量模式：精准匹配固定值
**匹配表达式的值与 case 后的字面量（字符串、整数、布尔值等）完全相等时命中，是最基础的模式。**
```python
# 匹配字符串
fruit = "apple"
match fruit:
    case "apple":
        print("苹果，单价 5 元/斤")  # 输出：苹果，单价 5 元/斤
    case "banana":
        print("香蕉，单价 3 元/斤")
    case "orange":
        print("橙子，单价 4 元/斤")
    case _:
        print("未知水果")

# 匹配布尔值（注意：True/False 是字面量，0/1 不匹配）
flag = True
match flag:
    case True:
        print("条件成立")  # 输出：条件成立
    case False:
        print("条件不成立")
```

#### <2>变量模式：匹配任意值并绑定到变量
**case 后跟变量名时，会匹配任意值，并将匹配表达式的值绑定到该变量（类似赋值），常用于接收不确定的值。**
```python
# 变量模式匹配任意值
user_input = "hello"
match user_input:
    case "quit":
        print("退出程序")
    case msg:  # 匹配任意值，绑定到 msg 变量
        print(f"你输入的内容：{msg}")  # 输出：你输入的内容：hello

# 结合多模式使用
num = 7
match num:
    case 1 | 3 | 5:
        print("奇数（1/3/5）")
    case x:  # 匹配其他所有值，绑定到 x
        print(f"其他数字：{x}")  # 输出：其他数字：7
```

#### <3>通配符模式： _  匹配任意值但不绑定
**_ 是特殊的通配符模式，可匹配任意值，但不会将值绑定到变量（即无法在代码块中使用该值），常用于「忽略无关值」或「兜底匹配」（类似 else）。**
```python
# 兜底匹配（类似 else）
color = "purple"
match color:
    case "red":
        print("红色")
    case "green":
        print("绿色")
    case "blue":
        print("蓝色")
    case _:  # 匹配所有未命中的情况
        print("其他颜色")  # 输出：其他颜色

# 忽略无关值（仅关心特定位置的值）
person = ("小明", 22, "男")  # 元组：(姓名, 年龄, 性别)
match person:
    case (name, _, gender):  # 忽略年龄（用 _ 匹配）
        print(f"姓名：{name}，性别：{gender}")  # 输出：姓名：小明，性别：男
```

#### <4>结构模式：匹配容器（列表/元组/字典）的结构
**可精准匹配列表、元组、字典的「结构」（长度、元素位置、键名），并提取其中的元素，是 match-case 最强大的功能之一。**
```python
# 1. 匹配元组结构（固定长度、提取元素）
point = (3, 4)  # 二维坐标：(x, y)
match point:
    case (0, 0):
        print("原点")
    case (x, 0):
        print(f"在 x 轴上，x = {x}")
    case (0, y):
        print(f"在 y 轴上，y = {y}")
    case (x, y):
        print(f"在平面内，坐标 ({x}, {y})")  # 输出：在平面内，坐标 (3, 4)

# 2. 匹配列表结构（支持可变长度，用 * 表示剩余元素）
nums = [1, 2, 3, 4]
match nums:
    case [1, *rest]:  # 以 1 开头，剩余元素绑定到 rest
        print(f"以 1 开头，剩余元素：{rest}")  # 输出：以 1 开头，剩余元素：[2, 3, 4]
    case [a, b]:
        print(f"两个元素：{a}, {b}")

# 3. 匹配字典结构（匹配指定键，提取值,不需要完全匹配，可以是超集）
user = {"name": "小红", "age": 19, "score": 92}
match user:
    case {"name": name, "age": age}:  # 匹配包含 name 和 age 键的字典，提取值
        print(f"姓名：{name}，年龄：{age}")  # 输出：姓名：小红，年龄：19
    case {"name": name, "score": score}:
        print(f"姓名：{name}，成绩：{score}")
```

#### <5>类型模式：匹配指定数据类型
**通过 “类型名()” 的形式匹配值的类型，可同时将值绑定到变量，适合「根据数据类型执行不同逻辑」的场景。**
```python
# 类型模式匹配
data = 3.14
match data:
    case int():
        print(f"整数类型：{data}")
    case float():
        print(f"浮点数类型：{data}")  # 输出：浮点数类型：3.14
    case str():
        print(f"字符串类型：{data}")
    case list():
        print(f"列表类型：{data}")
    case _:
        print("未知类型")

# 类型模式 + 绑定变量（更常用）
data = "hello"
match data:
    case int(n):
        print(f"整数：{n * 2}")
    case str(s):
        print(f"字符串：{s.upper()}")  # 输出：字符串：HELLO
    case float(num):
        print(f"浮点数：{num + 1}")
```

#### <6>守卫模式：在模式匹配的基础上添加额外条件
**通过 “if 条件” 给模式添加额外限制，仅当「模式匹配成功」且「条件成立」时才执行代码块，弥补了纯模式匹配的灵活性不足。**
```python
# 守卫模式（模式 + 额外条件）
num = 18
match num:
    case int(n) if n > 18:
        print(f"大于 18 的整数：{n}")
    case int(n) if n == 18:
        print(f"等于 18 的整数：{n}")  # 输出：等于 18 的整数：18
    case int(n) if n < 18:
        print(f"小于 18 的整数：{n}")

# 结合结构模式使用
point = (5, 5)
match point:
    case (x, y) if x == y:
        print(f"在 y = x 直线上，坐标 ({x}, {y})")  # 输出：在 y = x 直线上，坐标 (5, 5)
    case (x, y):
        print(f"坐标 ({x}, {y})")
```

### （3）case 分支的避坑要点
- **版本限制**
	- match-case 仅支持 Python 3.10 及以上版本，低版本使用会报语法错误；
- **自动终止**
	- **无需 break**：与其他语言（如 Java）的 switch-case 不同，Python 的 match-case 匹配成功后会自动终止，无需添加 break
	- **需继续匹配后续模式**：可使用 case ... if False 等特殊技巧，**但不推荐**；

- **通配符 _ 的位置**
	- **\_** ：是“兜底匹配”，需**放在所有 case 的最后**，否则会覆盖前面的模式（导致前面的模式无法命中）；

- **模式匹配的优先级**
	- 更**具体的模式**（如字面量模式、固定结构模式）应**放在前面**，更**通用的模式**（如变量模式、通配符）应**放在后面**；

- **字典模式的匹配规则**
	- 匹配字典时，仅要求字典包含 case 中指定的键即可（**不要求键完全一致**），会提取指定键的值；

- **避免过度使用**
	- 简单的条件判断（如单一比较）用 if 语句更直观
	- match-case 更适合多值精准匹配、结构匹配、类型匹配场景。

### （4）与其他语言switch的区别
- Python的match-case**不需要break语句**阻止继续执行（自动终止）
- 支持复杂的模式匹配而非仅支持常量值比较
- 可以直接从匹配中提取和绑定变量
- 支持结构解构和类型匹配

---

# 六、while循环

## 1.核心语法与基础机制
**while 循环的语法简洁，核心由「条件表达式」和「循环体」组成，通过条件的布尔值控制循环的启动与终止。**
``` python
# 基础语法
while 条件表达式:
    缩进的循环体代码  # 条件为 True 时执行
    （可选但推荐）条件更新语句  # 用于修改条件，避免无限循环

# 空循环（仅占位/延迟，无实际逻辑）
while 条件表达式:
    pass  # pass 是占位符，代表“无操作”（用与暂时不需要实现，后续补充/扩展）
```
```python
# 示例1：基本循环（输出 1-5）
num = 1  # 初始化条件变量
while num <= 5:  # 条件：num 小于等于 5
    print(num)  # 循环体：输出 num，依次为 1、2、3、4、5
    num += 1  # 条件更新：num 自增 1（关键！避免无限循环）

# 示例2：条件更新（延迟 3 秒，仅作演示）
import time
count = 0
while count < 3:
    time.sleep(1)  # 暂停 1 秒
    count += 1
    print(f"延迟 {count} 秒...")

# 示例3：纯占位空循环（代码框架搭建阶段）
# 作用：先定义循环结构，后续补充业务逻辑，避免语法错误
empty_flag = False  # 初始为False，循环不执行（仅占位）
while empty_flag:
    pass # while 必须有循环体，若没有任何实现又不用 pass 占位，则会报错


# 示例4：无限循环占位（需配合break，否则会卡死）
count = 0
while True:
    count += 1
    if count > 5:
        break
    pass  # 循环体逻辑后续补充

# 示例5：嵌套 while 循环（多层循环）
row = 1
while row <= 3:  # 外层循环：控制行数
    col = 1
    while col <= 4:  # 内层循环：控制每行的列数
        print("*", end=" ")  # end=" " 使打印不换行
        col += 1
    print()  # 一行打印完毕，换行
    row += 1
# 输出：
# * * * * 
# * * * * 
# * * * * 
```
**pass：「语法占位符」**
- **核心作用**：避免 “缺少语句导致的语法错误”

## 2.循环控制关键字：break 与 continue
**默认情况下，while 循环按「条件判断→循环体执行」的流程重复，但实际开发中常需灵活控制循环（如提前终止、跳过某次循环），此时需用到 break 和 continue 两个核心关键字。**

### （1）break（立即终止循环）
**作用**：当执行到 break 时，直接终止当前所在的循环，跳出循环体，程序执行循环之后的代码（无论条件是否仍成立）。
```python
# 示例：找到目标值后立即终止循环
target = 3
num = 1
while num <= 10:
    if num == target:
        print(f"找到目标值：{num}，终止循环")
        break  # 找到目标，立即终止循环
    print(f"当前值：{num}")
    num += 1
# 输出：当前值：1 → 当前值：2 → 找到目标值：3，终止循环
```

### （2）continue（跳过当前循环，进入下一轮）
**作用**：当执行到 continue 时，跳过当前循环体中剩余的代码，直接回到「条件判断」环节，准备执行下一轮循环（不会终止循环）。
```python
# 示例：跳过偶数，只输出奇数（1-5）
num = 0
while num < 5:
    num += 1	# 注：必须在 continue 之前更新循环变量！
    if num % 2 == 0:  # 判断是否为偶数
        continue  # 是偶数则跳过后续打印代码
    print(f"奇数：{num}")  # 仅输出奇数：1、3、5

# ❌ 错误示例：会导致死循环
num = 0
while num < 5:
    if num % 2 == 0:
        continue  # 当 num=0 时，跳过了 num+=1，永远卡在 0
    print(f"奇数：{num}")
    num += 1
```



# 七、用户输入（input）
## 1.核心语法与基础用法
**input() 函数的语法简洁，核心功能是“阻塞程序等待用户输入，获取输入内容并返回”。根据是否需要提示用户，可分为带提示/不带提示两种用法**
```python
# 基础语法
# 用法1：无提示信息（仅等待用户输入）
输入变量 = input()

# 用法2：带自定义提示信息（推荐，提升用户体验）
输入变量 = input("请输入xxx：")  # 提示信息会直接显示在控制台，引导用户输入

# 使用示例
# 示例1：无提示信息的输入
print("请输入任意内容，按回车结束：")
content = input()  # 程序阻塞，等待用户输入
print(f"你输入的内容是：{content}")  # 输出用户输入的内容

# 示例2：带提示信息的输入（日常开发首选）
name = input("请输入你的姓名：")
age = input("请输入你的年龄：")
print(f"欢迎你，{name}！你的年龄是 {age} 岁。")
```

## 2.input() 函数的核心特性（必记）
**理解 input() 的核心特性是正确处理用户输入的关键，尤其是“返回值类型”和“阻塞特性”，直接影响后续代码逻辑**
### （1）返回值永远是字符串类型
- **无论用户输入的是数字（如 18）、布尔值（如 True）还是符号（如 @），input() 的返回值都默认是 字符串（str）**
- **若需要用输入内容进行数学运算，必须“手动进行类型转换”，否则会报错**
```python
# 典型错误示例：直接用输入的“数字字符串”做运算
score = input("请输入你的成绩：")
print(score + 10)  # 报错：TypeError: can only concatenate str (not "int") to str

# 正确示例：先转换为对应数字类型
score = int(input("请输入你的成绩："))  # 转换为整数（适用于整数输入）
print(f"成绩+10后的分数：{score + 10}")  # 正常执行

height = float(input("请输入你的身高（m）："))  # 转换为浮点数（适用于小数输入）
print(f"身高的2倍：{height * 2}")  # 正常执行
```

### （2）程序阻塞特性
- **当程序执行到 input() 时，会立即暂停（阻塞），直到用户完成输入并按下「回车键」，程序才会继续执行后续代码**
- **注意：用户未输入时，程序会一直等待，不会自动继续。**

### （3）空输入的处理
- **若用户未输入任何内容，直接按下回车键，input() 会返回一个 空字符串（""）。**
- **空输入是常见场景，需根据业务需求判断是否允许，避免程序因空值出现逻辑错误。**
- **此外，input() 在遇到 EOF（如 Ctrl+D / Ctrl+Z 或重定向空文件）时会抛 EOFError，在某些自动化/测试场景下需要注意。**
```python
# 示例：处理空输入
user_input = input("请输入内容（不能为空）：")
if user_input == "":  # 判断是否为空输入
    print("错误：输入不能为空，请重新输入！")
else:
    print(f"你输入的内容是：{user_input}")
```

## 3.输入内容的类型转换（关键操作）
**日常开发中，用户输入常需作为数字（int/float）、布尔值（bool）等类型使用，因此“类型转换”是input() 用法的核心延伸。需根据输入场景选择合适的转换方式，并处理转换失败的情况。**

### （1）常见类型转换场景与示例
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110115257300.png)

### （2）多值输入的处理技巧
**若需要用户一次性输入多个值（如“姓名 年龄 成绩”“多个数字”），可结合 split() 方法拆分输入字符串，再批量转换类型：**
```python
# 示例1：用空格分隔多个值（如“小明 20 95”）
while True:
	info = input("请输入姓名 年龄 成绩（用空格分隔）：").split()
	if len(info) != 3:
	    print("错误：请输入恰好 3 个值！")
	else:
	    name = info[0]
	    age = int(info[1])
	    score = int(info[2])
	    print(f"姓名：{name}，年龄：{age}，成绩：{score}")
	    break

# 示例2：用逗号分隔多个值（如“90,85,92”）
scores_input = input("请输入多个成绩，用逗号分隔：").split(",")
scores = [int(s) for s in scores_input]  # 列表推导式批量转换为整数
print(f"成绩列表：{scores}，平均成绩：{sum(scores)/len(scores):.2f}")
```

## 4.核心难点：输入验证与异常处理
**用户输入具有不可控性（如要求输入数字却输入文字、输入超出范围的值），直接使用输入内容会导致程序崩溃。因此，“输入验证”和“异常处理”是保障程序稳定性的关键，常用 try-except 语句捕获转换异常。**
### （1）基础异常处理：确保输入可转换为目标类型
**当用户输入无法转换为目标类型（如输入“abc”却要转换为 int）时，会触发 ValueError 异常，用 try-except 捕获并提示用户重新输入**
```python
# 示例：确保用户输入的是整数
while True:  # 循环直到输入正确
    try:
        num = int(input("请输入一个整数："))
        print(f"你输入的整数是：{num}")
        break  # 输入正确，退出循环
    except ValueError:
        print("错误：输入的不是有效整数，请重新输入！")

```

### （2）进阶验证：同时验证类型与值的范围
**除了类型验证，还需验证输入值是否符合业务范围（如年龄 0-120、成绩 0-100），结合 if-else 实现完整验证逻辑**
```python
# 示例：验证输入的是 0-100 之间的成绩
while True:
    try:
        score = int(input("请输入成绩（0-100）："))
        # 验证值的范围
        if 0 <= score <= 100:
            print(f"成绩录入成功：{score}")
            break
        else:
            print("错误：成绩必须在 0-100 之间，请重新输入！")
    except ValueError:
        print("错误：输入的不是有效数字，请重新输入！")
```
### （3）复杂验证：多条件组合验证
**针对更复杂的输入需求（如密码长度、格式验证），可组合多个条件实现精准验证**
```python
# 示例：验证密码格式（长度≥6，包含字母和数字）
while True:
    password = input("请设置密码（长度≥6，包含字母和数字）：")
    # 验证条件：长度≥6、包含字母、包含数字
    if len(password) < 6:
        print("错误：密码长度必须≥6，请重新设置！")
    elif not any(c.isalpha() for c in password):
        print("错误：密码必须包含字母，请重新设置！")
    elif not any(c.isdigit() for c in password):
        print("错误：密码必须包含数字，请重新设置！")
    else:
        print("密码设置成功！")
        break
```

## 5.高级技巧：提升输入交互体验
**在基础用法之上，可通过一些技巧优化用户输入体验，适配更多特殊场景**
### （1）隐藏敏感输入（如密码）
**默认情况下，用户输入的内容会明文显示在控制台，对于密码等敏感信息，可使用 getpass 模块的 getpass() 函数隐藏输入（输入内容不显示）**
```python 
import getpass  # 导入模块

# 隐藏密码输入（输入内容不显示）
username = input("请输入用户名：")
password = getpass.getpass("请输入密码：")

# 简单验证（实际开发中需对接数据库）
if username == "admin" and password == "123456":
    print("登录成功！")
else:
    print("用户名或密码错误！")
```

### （2）预设默认值（用户可直接回车使用默认值）
**对于可选输入项，可设置默认值，用户直接按下回车键时，使用预设值，提升交互效率**
```python
# 示例：设置默认年龄为 18
while True:
	try:
		user_input = input("请输入你的年龄（默认 18）：")
		age = int(user_input) if user_input != "" else 18 # 空输入则使用默认值 18
		break
	except ValueError:
		print("错误：输入的不是有效数字，请重新输入！")
print(f"你的年龄：{age}")
```

## 6.避坑要点与注意事项
- **永远不要信任用户输入**：用户输入具有不可控性，必须进行验证（类型、范围、格式），否则会导致程序崩溃或逻辑错误
- **牢记返回值是字符串**：无论用户输入什么，先按字符串处理，再根据需求转换类型，避免直接用输入内容做运算
- **循环验证时注意退出条件**：用 while True 做循环验证时，必须确保有 break语句退出循环，避免无限循环
- **区分 split() 与 split('分隔符')**
	- **split()**： 默认按任意空白字符（空格、制表符、换行）拆分
	- **split('分隔符')**：按指定分隔符拆分，需根据用户输入格式选择
- **敏感输入用 getpass 而非 input**：密码、验证码等敏感信息，不要用 input() 明文显示，优先使用 getpass.getpass()
- **兼容空输入场景**：用户可能直接回车，需提前判断空字符串并处理（如设置默认值、提示重新输入）


# 八、函数
## 1.函数的核心定义
**函数是 Python 中「封装可复用逻辑的基本单元」，本质是“输入→处理→输出”的黑盒**
- **输入**：通过参数接收外部数据
- **处理**：执行封装的代码逻辑
- **输出**：通过返回值向外部反馈结果（无返回值则默认返回 None）

**注！！**：函数体必须**通过缩进来标识**（默认 4 个空格），这是 Python 区分代码块归属的核心规则。

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110153245076.png)
```python
# 完整结构：关键字 + 函数名 + 参数 + 冒号 + 缩进体 + 返回值
def 函数名(参数列表):
    """文档字符串（可选，用于说明函数功能）"""
    # 函数体（要执行的逻辑）
    处理逻辑
    return 返回值  # 可选，无return则返回None

# 最简示例：计算两数之和
def add(a, b):
    """计算两个数的和"""
    result = a + b
    return result

# 调用函数（执行函数逻辑，获取返回值）
sum_result = add(10, 20)
print(sum_result)  # 输出：30
```

## 2.函数的参数体系（核心重点）
**参数组合顺序**:位置参数 → 默认参数 → \*args → 仅限关键字参数 → \*\*kwargs
### （1）位置参数
- **定义**：最基础的参数，调用时**必须按「定义顺序」传入**，**数量必须匹配**
- **特点**：无默认值，不传则报错
```python
def print_info(name, age):  # name、age是位置参数
    print(f"姓名：{name}，年龄：{age}")

# 正确调用（按位置传参）
print_info("张三", 25)  # 输出：姓名：张三，年龄：25
# 错误调用（少传参数）
# print_info("李四")  # 报错：missing 1 required positional argument: 'age'
```

### （2）默认参数
- **定义**：给参数指定默认值，调用时**可省略不传**（使用默认值）
- **特点**
	- 默认参数**必须放在「位置参数之后」**
	- 默认参数**建议用「不可变对象」**（如数字、字符串、元组），避免用列表/字典（可变对象会导致多次调用共享数据）
```python
# city是默认参数（默认值"北京"），必须放在位置参数后
def print_info(name, age, city="北京"):
    print(f"姓名：{name}，年龄：{age}，城市：{city}")

# 省略默认参数（使用"北京"）
print_info("张三", 25)  # 输出：姓名：张三，年龄：25，城市：北京
# 覆盖默认参数（传"上海"）
print_info("李四", 30, "上海")  # 输出：姓名：李四，年龄：30，城市：上海

# 坑点：默认参数用可变对象（列表）的问题
def add_item(item, lst=[]):  # 不推荐！lst是可变对象
    lst.append(item)
    return lst

print(add_item(1))  # [1]（第一次调用，lst初始化为空列表）
print(add_item(2))  # [1,2]（第二次调用，lst复用了上次的列表）
# 正确写法：默认参数用None，内部初始化
def add_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
print(add_item(1))  # [1]
print(add_item(2))  # [2]
```

### （3）关键字参数
- **定义**：调用函数时，通过 **「参数名=值」** 的方式传参，**无需按位置顺序传入**
- **特点**：提高代码可读性，适合参数多的场景
- **注意**：调用时，关键字参数不能在位置参数前
```python
def print_info(name, age, city):
    print(f"姓名：{name}，年龄：{age}，城市：{city}")

# 关键字参数（顺序可打乱）
print_info(age=25, name="张三", city="北京")  # 输出正常
# 混合位置参数+关键字参数（位置参数必须在前）
print_info("张三", city="上海", age=30)  # 正确
# print_info(city="广州", "李四", 28)  # 错误：关键字参数不能在位置参数前
```

### （4）仅限关键字参数（ \* ）（Python 3+）
- **定义**：必须通过关键字参数传入的参数，放在 \*args 之后、\*\*kwargs 之前；
- **作用**：强制调用者明确参数名，避免歧义
- **定义仅限关键字**：
    - 1.使用 **\*** 作为分隔符：
        \* 之后的参数都被标记为仅限关键字参数，调用时必须用 关键字=值 的形式传递。
    - 2.\*args 之后的参数自动成为仅限关键字参数
```python
# 1.使用 * 作为分隔符
def func(a, *, b, c):  # * 之后的b、c是仅限关键字参数
    print(a, b, c)

func(1, b=2, c=3)  # 正确（必须用关键字传b、c）
# func(1, 2, 3)  # 错误：b,c must be keyword arguments

## 2.\*args 之后
def print_info(*args, sep=' ', end='\n', file=None):
    """
    模拟Python内置的print函数：*args接收任意位置参数，sep/end/file是仅限关键字参数
    """
    output = sep.join(map(str, args))
    print(output, end=end, file=file)

# ✅ 正确调用
print_info('Hello', 'Python', sep='-', end='!\n')  # 输出: Hello-Python!

# ❌ 错误调用
# print_info('Hello', 'Python', '-', '!\n')  # sep/end被当作args传递，达不到预期效果
```

### （5）万能参数（\*args 和 \*\*kwargs）
- **用于接收「不确定数量」的参数，是实现通用函数的核心**
- **注意**：\*args 和 \*\*kwargs 均**支持不传入任何对应参数**，程序不会报错，会自动以空容器接收

#### <1>\*args：接收任意数量的位置参数，打包为“元组”
```python
def sum_all(*args):  # args是元组，接收所有位置参数
    print("args类型：", type(args))  # <class 'tuple'>
    total = 0
    for num in args:
        total += num
    return total

# 传入任意数量参数
print(sum_all(1, 2, 3))  # 6
print(sum_all(10, 20, 30, 40))  # 100
print(sum_all())  # 0（无参数时args是空元组）

# 解包列表/元组传入args
nums = [1,2,3]
print(sum_all(*nums))  # 6（*表示解包）
```

#### <2>\*\*kwargs：接收任意数量的关键字参数，打包为“字典”
```python
def print_kwargs(**kwargs):  # kwargs是字典，接收所有关键字参数
    print("kwargs类型：", type(kwargs))  # <class 'dict'>
    for k, v in kwargs.items():
        print(f"{k}: {v}")

# 传入任意数量关键字参数
print_kwargs(name="张三", age=25, city="北京")
# 输出：
# name: 张三
# age: 25
# city: 北京

# 解包字典传入kwargs
info = {"name": "李四", "age": 30}
print_kwargs(**info)  # **表示解包字典
```

#### <3>\*args 与 \*\*kwargs 四种传参场景演示
```python
# 定义支持任意位置参数和关键字参数的函数
def func(*args, **kwargs):
    print(f"1. 位置参数 args：{args}")  # args打包为元组
    print(f"2. 关键字参数 kwargs：{kwargs}")  # kwargs打包为字典
    print("-" * 30)  # 分隔线，清晰区分不同场景

# 场景1：不传参
print("场景1：不传参")
func()

# 场景2：仅*args传参（位置参数）
print("场景2：仅*args传参（2个位置参数）")
func(10, 20)  # 直接传位置参数，自动打包到args
# 也可通过解包可迭代对象传参（呼应前文*解包知识）
print("场景2扩展：解包列表传参给*args")
func(*[30, 40])  # 解包列表为位置参数，等价于func(30,40)

# 场景3：仅**kwargs传参（关键字参数）
print("场景3：仅**kwargs传参（2个关键字参数）")
func(name="Python", version=3.10)  # 直接传关键字参数，自动打包到kwargs
# 也可通过解包字典传参（呼应前文**解包知识）
print("场景3扩展：解包字典传参给**kwargs")
func(**{"name": "Java", "version": 17})  # 解包字典为关键字参数，等价于func(name="Java", version=17)

# 场景4：*args + **kwargs 混合传参
print("场景4：混合传参（2个位置参数 + 2个关键字参数）")
func(1, 2, x=3, y=4)
# 混合解包传参（实用场景）
print("场景4扩展：混合解包传参")
func(*(5, 6), **{"x":7, "y":8})  # 解包元组+字典，等价于func(5,6,x=7,y=8)
```

#### <4>参数组合顺序（必须遵守）
**位置参数 → 默认参数 → \*args → 仅限关键字参数 → \*\*kwargs**
```python
# 正确顺序示例
def func(a, b=10, *args, c, **kwargs):
    print(a, b, args, c, kwargs)

func(1, 2, 3, 4, c=5, d=6, e=7)  # 输出：1 2 (3, 4) 5 {'d': 6, 'e': 7}
```

## 3.函数的返回值
### （1）基本规则
- **无 return 语句**：函数执行完默认返回 None
- **有 return 语句**：执行到 return 立即终止函数，返回指定值
- **可返回任意类型**：数字、字符串、列表、字典、函数、对象等

### （2）多返回值（Python 特色）
**Python 没有真正的“多返回值”，本质是返回「元组」，只是省略了括号**
```python
def get_info():
    name = "张三"
    age = 25
    return name, age  # 等价于 return (name, age)

# 接收多返回值（自动解包）
name, age = get_info()
print(name, age)  # 张三 25

# 也可以接收为元组
result = get_info()
print(result)  # ('张三', 25)
```

## 4.函数的作用域（变量可见性）
### （1）四种作用域介绍
Python 遵循「LEGB 规则」，变量查找优先级：
- **全局作用域（Global）**
	- 模块（.py文件）级别的变量，可被当前模块内的所有函数、类访问，除非被局部变量遮蔽；
- **局部作用域（Local）**
	- 函数内部定义的变量，仅在当前函数体内有效，函数执行结束后变量被销毁
- **嵌套作用域（Enclosing）**
	- 外部函数（嵌套函数的外层）定义的变量，仅对外部函数自身及内部嵌套函数可见
- **内置作用域（Built-in）**
	- Python 内置的函数、关键字等（如 print、len、int），无需定义即可在任意位置直接使用。

### （2）四种作用域示例
#### <1>全局作用域示例
全局变量定义在模块级别，可被模块内所有函数访问，**修改需用 global 声明**
```python
# 全局变量：定义在模块最外层，模块内所有函数可访问
global_app_name = "Python学习工具"
global_user_count = 0

def register_user(username):
    # 访问全局变量global_app_name（无需声明，直接访问）
    print(f"【{global_app_name}】用户注册：{username}")
    # 修改全局变量需用global声明
    global global_user_count
    global_user_count += 1

def get_user_count():
    # 不同函数可共享全局变量
    print(f"【{global_app_name}】当前注册用户数：{global_user_count}")

# 调用函数，共享并修改全局变量
register_user("张三")  # 输出：【Python学习工具】用户注册：张三
register_user("李四")  # 输出：【Python学习工具】用户注册：李四
get_user_count()       # 输出：【Python学习工具】当前注册用户数：2

# 模块外部（当前脚本内）可直接访问全局变量
print(f"全局应用名：{global_app_name}")  # 输出：全局应用名：Python学习工具
```

#### <2>局部作用域示例
局部变量仅在定义它的函数内部有效，外部无法访问，函数执行完毕后变量释放
```python
# 定义函数，内部包含局部变量
def calculate_area(radius):
    # 局部变量：仅在calculate_area内部有效
    pi = 3.14159  
    area = pi * radius ** 2
    print(f"圆的面积：{area}")  # 函数内部可正常访问局部变量pi、area

# 调用函数
calculate_area(5)  # 输出：圆的面积：78.53975

# 错误：外部无法访问函数内的局部变量
# print(pi)  # 报错：name 'pi' is not defined
# print(area)  # 报错：name 'area' is not defined
```

#### <3>嵌套作用域示例
嵌套作用域变量由外部函数定义，内部函数可直接访问，外部函数外无法访问
```python
# 外部函数（定义嵌套作用域变量）
def outer_func(company):
    # 嵌套作用域变量：对outer_func和inner_func可见
    department = "技术部"  
    
    # 内部函数（嵌套在outer_func内部）
    def inner_func(name):
        # 可直接访问嵌套作用域变量department和外部函数参数company
        print(f"姓名：{name}，公司：{company}，部门：{department}")
    
    inner_func("张三")  # 内部函数调用，正常访问嵌套变量

# 调用外部函数
outer_func("字节跳动")  # 输出：姓名：张三，公司：字节跳动，部门：技术部

# 错误：外部无法访问嵌套作用域变量
# print(department)  # 报错：name 'department' is not defined
# 错误：外部无法直接调用内部函数（间接体现嵌套作用域隔离）
# inner_func("李四")  # 报错：name 'inner_func' is not defined
```

#### <4>内置作用域示例
内置作用域包含 Python 自带的函数/关键字，无需定义即可直接使用，**优先级最低（当其他作用域有同名变量时会被遮蔽）**
```python
# 直接使用内置函数print（无需定义，属于内置作用域）
print("使用内置函数print输出")  # 输出：使用内置函数print输出

# 直接使用内置函数len（计算字符串长度）
str_data = "Python作用域"
print(f"字符串长度：{len(str_data)}")  # 输出：字符串长度：8

# 注意：同名局部变量会遮蔽内置函数
def test_builtin():
    # 定义与内置函数同名的局部变量，遮蔽内置len
    len = 10  
    print(f"局部变量len：{len}")  # 输出：局部变量len：10
    # 此时无法直接使用内置len函数，需通过__builtins__访问
    print(f"内置len计算长度：{__builtins__.len('test')}")  # 输出：内置len计算长度：4

test_builtin()

# 外部仍可正常使用内置len（局部变量不影响全局/内置作用域）
print(f"外部使用内置len：{len('hello')}")  # 输出：外部使用内置len：5
```

### （3）global 与 nonlocal 关键字
- 在 Python 作用域规则中，默认**“内部作用域不能直接修改外部作用域变量”**。
- global 和 nonlocal 关键字的**核心作用**：「打破这一限制」，明确声明变量的作用域归属，实现对外部变量的修改。
- 两者适用场景不同，需严格区分使用。

#### <1>global 关键字
- **作用**:
	- 声明变量是「全局作用域」的变量，允许**在局部作用域（函数内部）中修改全局变量**
- **核心规则**：
	- **仅读取全局变量**：无需用 global 声明，直接访问即可
	- **在函数内部修改全局变量**：**必须先通过 global 声明**，否则会被解析为局部变量
	- 可同时声明多个全局变量，用逗号分隔（如 global a, b, c）
```python
# 全局变量（模块级别）
global_count = 0
global_name = "全局默认名"

# 示例1：仅读取全局变量（无需global声明）
def read_global():
    # 仅访问全局变量，不修改，无需声明global
    print(f"仅读取全局变量：count={global_count}, name={global_name}")

read_global()  # 输出：仅读取全局变量：count=0, name=全局默认名

# 示例2：修改全局变量（必须用global声明）
def modify_global():
    # 声明要修改的是全局变量（必须在赋值前声明）
    global global_count, global_name
    
    # 修改全局变量
    global_count += 10
    global_name = "修改后的全局名"
    
    # 读取全局变量（声明后可直接使用）
    print(f"函数内访问全局变量：count={global_count}, name={global_name}")

# 调用函数，触发全局变量修改
modify_global()  # 输出：函数内访问全局变量：count=10, name=修改后的全局名

# 函数外部验证全局变量已被修改
print(f"函数外访问全局变量：count={global_count}, name={global_name}")  # 输出：count=10, name=修改后的全局名

# 反面案例：未声明global直接赋值（被解析为局部变量）
def wrong_modify():
    global_count = 100  # 此处是新的局部变量，与全局变量无关
    print(f"函数内局部变量：{global_count}")

wrong_modify()  # 输出：函数内局部变量：100
print(f"全局变量未变：{global_count}")  # 输出：全局变量未变：10
```

#### <2>nonlocal 关键字
- **作用**：
	- 声明变量是「嵌套作用域」的变量（外部函数的变量），允许**在内部函数中修改外部函数的变量**
- **核心规则**：
	- nonlocal 仅适用于嵌套结构，**不能用于修改全局变量**
	- 声明的变量**必须在外部函数中已定义**，不能是未定义的新变量
	- 多层嵌套 nonlocal逐层向上查找变量
	- 可同时声明多个嵌套变量，用逗号分隔（如 nonlocal x, y）
```python
# 示例1：仅读取外部函数变量（无需nonlocal声明）
def outer_func_read():
    # 外部函数变量（嵌套作用域）
    outer_var = "外部函数的变量"
    
    def inner_func_read():
        # 仅读取外部变量，不修改，无需声明nonlocal
        print(f"内部函数仅读取外部变量：{outer_var}")
    
    inner_func_read()  # 调用内部函数

outer_func_read()  # 输出：内部函数仅读取外部变量：外部函数的变量

# 示例2：修改外部函数变量（必须用nonlocal声明）
def outer_func():
    # 外部函数变量（嵌套作用域）
    outer_num = 10
    outer_str = "外部默认字符串"
    
    def inner_func():
        # 声明要修改的是嵌套作用域变量
        nonlocal outer_num, outer_str
        
        # 修改嵌套作用域变量
        outer_num *= 2
        outer_str = "修改后的嵌套字符串"
        
        print(f"内部函数访问：num={outer_num}, str={outer_str}")
    
    inner_func()  # 调用内部函数
    print(f"外部函数验证：num={outer_num}, str={outer_str}")  # 变量已被内部函数修改

outer_func()
# 输出：
# 内部函数访问：num=20, str=修改后的嵌套字符串
# 外部函数验证：num=20, str=修改后的嵌套字符串

# 反面案例：nonlocal修饰未定义的变量
def wrong_outer():
    def wrong_inner():
        nonlocal undefined_var  # 语法错误：nonlocal声明的变量必须在外层函数作用域中存在
        undefined_var = 5
    wrong_inner()

# 注意：这段代码在**定义阶段**就会触发 SyntaxError（语法错误），而非运行时错误
# wrong_outer()  # 实际执行前就会报错：SyntaxError: no binding for nonlocal 'undefined_var' found

# 示例3：多层嵌套场景（nonlocal逐层向上查找变量）
def outer_layer1():
    var = "最外层变量"  # 层级1：最外层外部函数变量
    
    def outer_layer2():  # 层级2：中间嵌套函数
        def inner_layer():  # 层级3：最内层内部函数
            nonlocal var  # 查找规则：从外层2开始向上找，最终找到outer_layer1的var
            var = "被最内层函数修改"
            print(f"最内层函数修改后：{var}")
        
        inner_layer()  # 调用最内层函数
        print(f"中间层函数访问：{var}")  # 能访问到被修改后的var
    
    outer_layer2()  # 调用中间层函数
    print(f"最外层函数验证：{var}")  # 最终变量被修改

outer_layer1()
# 输出：
# 最内层函数修改后：被最内层函数修改
# 中间层函数访问：被最内层函数修改
# 最外层函数验证：被最内层函数修改

# 示例4：多层嵌套中存在同名变量（nonlocal优先找最近的外层变量）
def outer1():
    var = "outer1的变量"
    
    def outer2():
        var = "outer2的变量"  # 与外层同名的中间层变量
        
        def inner():
            nonlocal var  # 优先找最近的外层（outer2）的var，而非outer1的var
            var = "修改outer2的变量"
            print(f"内层函数：{var}")
        
        inner()
        print(f"outer2函数：{var}")  # 被修改的是outer2的var
    
    outer2()
    print(f"outer1函数：{var}")  # outer1的var未被修改

outer1()
# 输出：
# 内层函数：修改outer2的变量
# outer2函数：修改outer2的变量
# outer1函数：outer1的变量
```

#### <3>global 与 nonlocal 核心差异对比
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110210400109.png)

## 5.内部函数（嵌套函数）
内部函数指「在另一个函数（外部函数）内部定义的函数」，也叫嵌套函数。它是实现闭包、装饰器等进阶特性的基础，核心价值是**封装内部逻辑、隔离作用域**。

### 核心定义与特征
- **定义位置**：必须在外部函数的函数体内部
- **作用域隔离**：内部函数默认只能在外部函数内部被调用，外部无法直接访问
- **变量访问**：内部函数可直接访问外部函数的变量（嵌套作用域变量），若需修改需用 nonlocal 声明
- **独立性**：**内部函数可作为外部函数的返回值传出，或作为参数传递给其他函数**（依托函数是一等对象的特性）。
```python
# 示例1：作用域隔离（内部函数变量外部不可访问）
def outer1():
    inner_var = "内部函数变量"  # 内部函数局部变量
    
    def inner1():
        print(inner_var)  # 内部可访问自身变量
    
    inner1()

outer1()
# print(inner_var)  # 报错：name 'inner_var' is not defined（外部无法访问，体现隔离）

# 示例2：变量访问与修改（内部函数访问/修改外层变量）
def outer2():
    outer_var = "外层变量"
    
    # 子示例2-1：仅访问外层变量（无需nonlocal）
    def inner2_read():
        print(f"仅访问外层变量：{outer_var}")
    
    # 子示例2-2：未用nonlocal修改（实际创建局部变量，修改失败）
    def inner2_wrong_modify():
        outer_var = "内部创建的局部变量"  # 与外层变量同名，不影响外层
        print(f"未用nonlocal修改后（局部变量）：{outer_var}")
    
    # 子示例2-3：用nonlocal修改（成功修改外层变量）
    def inner2_correct_modify():
        nonlocal outer_var  # 声明变量来自嵌套作用域
        outer_var = "修改后的外层变量"
        print(f"使用nonlocal修改后（外层变量）：{outer_var}")
    
    inner2_read()
    inner2_wrong_modify()
    print(f"未用nonlocal后，外层变量仍为：{outer_var}")
    inner2_correct_modify()
    print(f"使用nonlocal后，外层变量变为：{outer_var}")

outer2()
# 输出：
# 仅访问外层变量：外层变量
# 未用nonlocal修改后（局部变量）：内部创建的局部变量
# 未用nonlocal后，外层变量仍为：外层变量
# 使用nonlocal修改后（外层变量）：修改后的外层变量
# 使用nonlocal后，外层变量变为：修改后的外层变量

# 示例3：独立性（内部函数可作为返回值独立传递）
def outer3():
    def inner3():
        return "内部函数执行结果"
    
    return inner3  # 返回内部函数对象

# 独立调用返回的内部函数（与原外部函数脱离仍可执行）
inner_obj = outer3()
print(inner_obj())  # 输出：内部函数执行结果
```

## 6.\* 解包（序列解包）
- **在函数参数相关操作中，\* 除了用于定义可变长度参数（\*args），还常用于「序列解包」**
	- 将**可迭代对象**（列表、元组、字符串等）的**元素逐个拆解**，作为**独立参数**传入函数
	- 这是 Python 中简化参数传递的实用技巧，与**可变长度参数**相辅相成

### （1）核心作用
- 将**可迭代对象**“打散”为**单个元素**，避免手动逐个提取元素传参
- 可与普通参数、关键字参数混合使用，灵活适配不同函数的参数要求
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110162021473.png)

### （2）适用场景与示例
#### 场景1：解包列表/元组传入普通函数（需匹配参数数量）
```python
# 普通函数（2个位置参数）
def add(a, b):
    return a + b

# 列表解包
nums_list = [3, 5]
print(add(*nums_list))  # 等价于 add(3, 5)，输出：8

# 元组解包
nums_tuple = (4, 6)
print(add(*nums_tuple))  # 等价于 add(4, 6)，输出：10
```

#### 场景2：解包序列传入\*args（补充可变长度参数）
```python
# 可变长度参数函数
def sum_all(*args):
    total = 0
    for num in args:
        total += num
    return total

# 示例1：解包列表补充参数
base_nums = [1, 2, 3]
print(sum_all(*base_nums, 4, 5))  # 等价于 sum_all(1,2,3,4,5)，输出：15

# 示例2：解包字符串（字符串是可迭代对象，拆解为单个字符）
str_chars = "123"
print(sum_all(*str_chars))  # 等价于 sum_all('1','2','3')？注意：此处会报错，需确保元素类型匹配
# 正确用法：先转换元素类型
str_nums = "123"
print(sum_all(*map(int, str_nums)))  # 等价于 sum_all(1,2,3)，输出：6
```

#### 场景3：与关键字参数混合使用（解包需在关键字参数前）
```python
def print_info(name, age, city):
    print(f"姓名：{name}，年龄：{age}，城市：{city}")

# 解包列表传入前两个位置参数，第三个参数用关键字传
info_list = ["张三", 25]
print_info(*info_list, city="北京")  # 等价于 print_info("张三",25,city="北京")，输出正常

# 错误用法：解包不能在关键字参数后
# print_info(name="李四", *[30, "上海"])  # 报错：positional argument follows keyword argument
```
**位置参数 → 默认参数 → \*args → 仅限关键字参数 → \*\*kwargs**

## 7.\*\* 解包（字典解包）
**与 \* 解包序列对应，\*\* 专门用于「字典解包」** 
- 将字典的键值对拆解为“**参数名=参数值**”的关键字参数形式传入函数或“合并多个字典”
- 这一技巧常用于灵活**传递多个关键字参数**，尤其适合参数数量不确定或需要批量传入配置信息的场景。
- **注**：**仅字典可使用 \*\* 解包**

### （1）核心作用
- 将字典“打散”为多个关键字参数，避免手动逐个写 参数名=值，简化多关键字参数的传递
- 可与普通位置参数、关键字参数混合使用，精准匹配函数的参数要求
- 与 \*\*kwargs （接收可变关键字参数）相辅相成，实现“批量传参+动态接收”的组合。
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110201433413.png)

### （2）适用场景与示例
#### 场景1：解包字典传入普通函数（需字典键与函数参数名完全匹配）
```python
# 普通函数（3个位置/关键字参数）
def print_info(name, age, city):
    print(f"姓名：{name}，年龄：{age}，城市：{city}")

# 字典解包（键必须与参数名一致）
info_dict = {"name": "张三", "age": 25, "city": "北京"}
print_info(**info_dict)  # 等价于 print_info(name="张三", age=25, city="北京")，输出正常

# 错误用法：字典键与参数名不匹配
wrong_dict = {"name": "李四", "age": 30, "address": "上海"}
# print_info(**wrong_dict)  # 报错：got an unexpected keyword argument 'address'
```

#### 场景2：解包字典传入\*\*kwargs（补充可变关键字参数）
```python
# 可变关键字参数函数
def print_details(**kwargs):
    for k, v in kwargs.items():
        print(f"{k}：{v}")

# 解包字典补充参数
base_info = {"name": "王五", "age": 28}
print_details(**base_info, city="广州", gender="男")  # 等价于 print_details(name="王五", age=28, city="广州", gender="男")
# 输出：
# name：王五
# age：28
# city：广州
# gender：男
```

#### 场景3：混合\*解包与\*\*解包（序列+字典联合传参）
```python
# 混合参数的函数
def func(a, b, c, d):
    print(a + b + c + d)

# *解包列表（位置参数） + **解包字典（关键字参数）
list_data = [1, 2]
dict_data = {"c": 3, "d": 4}
func(*list_data, **dict_data)  # 等价于 func(1, 2, c=3, d=4)，输出：10

# 注意顺序：*解包（位置参数）必须在**解包（关键字参数）之前
# func(**dict_data, *list_data)  # 报错：positional argument follows keyword argument
```

### （3）注意事项
- **字典的键必须是「字符串类型」**，且**与函数的参数名完全一致**，否则会报“意外的关键字参数”错误
- 单个函数调用中可多次解包字典，但需确保不重复传递同名参数（否则报错 duplicate keyword argument）；

- **\*\* 仅用于解包「字典类型」**：dict、OrderedDict 等）
	- 不能用于列表、元组等序列（会报错 TypeError: __dict__ must be a mapping）；

- **\*\* 解包只能用于函数参数传递场景，不能单独用于赋值**
	- 如 a, b, c = \*\*dict_data 是错误的）；
- 混合传参时，必须遵循“位置参数→\*解包（位置参数）→关键字参数→\*\*解包（关键字参数）”的顺序。


## 8.函数的进阶特性
### （1）函数是一等对象（Python 核心特性）
**函数可以作为：参数、返回值、赋值给变量、存储在容器中（列表/字典），这是高阶函数的基础**
```python
# 1. 函数赋值给变量
def add(a, b):
    return a + b

f = add  # 变量f指向add函数
print(f(10, 20))  # 30

# 2. 函数作为参数传入另一个函数（高阶函数）
def calculate(func, x, y):
    return func(x, y)

print(calculate(add, 5, 6))  # 11（add作为参数传入）

# 3. 函数作为返回值（高阶函数）
def get_operation(op):
    if op == "+":
        return add
    elif op == "-":
        def sub(a, b):  # 内部函数
            return a - b
        return sub

f = get_operation("+")
print(f(3, 2))  # 5
f = get_operation("-")
print(f(3, 2))  # 1

# 4. 函数存储在列表中
funcs = [add, lambda x,y: x*y]
print(funcs[0](2,3))  # 5
print(funcs[1](2,3))  # 6
```

### （2）匿名函数（lambda）
- **定义**：
	- 用 **lambda 关键字**创建的极简函数，**无函数名（匿名）**，仅适用于简单逻辑
- **语法**：
	- lambda 参数列表: 表达式（表达式结果就是返回值，无 return）
- **返回值**：
	- lambda 匿名函数**一定有返回值**，「表达式」的计算结果会自动作为返回值
- **适用场景**
	- 临时使用的简单函数（如配合 map/filter/sorted）
```python
# 基础用法
add = lambda a, b: a + b
print(add(1, 2))  # 3

# 配合map使用（批量计算）
nums = [1,2,3]
result = list(map(lambda x: x*2, nums))
print(result)  # [2,4,6]

# 配合sorted使用（自定义排序）
students = [("张三", 25), ("李四", 20), ("王五", 30)]
# 按年龄升序排序（取元组第二个元素）
sorted_students = sorted(students, key=lambda x: x[1])
print(sorted_students)  # [('李四',20), ('张三',25), ('王五',30)]
```
```python
# 一定有返回值
# 1. 常规表达式，返回计算结果
add = lambda a, b: a + b
print(add(1,2))  # 输出 3（返回 a+b 的结果）

# 2. 无参数，返回固定值
hello = lambda: "hello"
print(hello())  # 输出 "hello"

# 3. 表达式结果为 None（仍有返回值）
empty = lambda x: x if x > 0 else None
print(empty(-5))  # 输出 None（并非没有返回值）
```

### （3）闭包
- **定义**：
	- 内部函数引用外部函数的变量，且外部函数返回内部函数（形成“封闭环境”）；
- **核心**：
	- **保存外部函数的变量状态**，即使外部函数执行完毕，变量仍可被内部函数访问；
```python
# 示例1
def outer(num1):
    # 外部函数变量
    def inner(num2):
        # 内部函数引用外部变量num1（闭包）
        return num1 + num2
    # 返回内部函数
    return inner

# 创建闭包实例（绑定num1=10）
f = outer(10)
# 调用内部函数（仍能访问num1=10）
print(f(5))  # 15
print(f(6))  # 16

# 示例2：“状态保持”
def counter():
    count = 0
    def increment():
        nonlocal count
        count += 1
        return count
    return increment

c = counter()
print(c())  # 1
print(c())  # 2
print(c())  # 3（count 状态被保持）

# ❌ 忘记使用 nonlocal 会报错
def counter_wrong():
    count = 0
    def increment():
        count += 1  # UnboundLocalError: 'count' referenced before assignment
        return count
    return increment
```

### （4）装饰器（闭包的核心应用）
#### <1>介绍
- **定义**：
	- 在不修改原函数代码的前提下，为函数添加额外功能（如日志、计时、权限校验）
- **核心原理**：
	- 装饰器本质是「高阶函数」（接收函数作为参数、返回函数作为结果），利用函数是一等对象的特性，通过“包装”原函数实现功能扩展，最终替换原函数的引用。
- 可使用 **@语法糖**简化包装过程
- **注！：**装饰器会导致原函数的 __name__、__doc__ 丢失，实际开发中通常会加 **@wraps**

```python
# 1. 定义装饰器函数（logger_decorator是装饰器，接收原函数作为参数）
def logger_decorator(func):  # func：接收被装饰的原函数（如后续的add函数）
    # 2. 定义内部包装函数（wrapper是核心，负责添加额外功能+执行原函数）
    # *args, **kwargs：万能参数，确保能接收原函数的任意参数，保证装饰器通用性
    def wrapper(*args, **kwargs):
        # 3. 前置额外功能：在原函数执行前执行（此处是打印调用日志）
        print(f"调用函数：{func.__name__}，参数：{args}, {kwargs}")
        # 4. 执行原函数：调用传入的原函数func，传入接收的参数，获取返回值
        result = func(*args, **kwargs)  # 若原函数无返回值，result为None
        # 5. 后置额外功能：在原函数执行后执行（此处是打印返回值）
        print(f"函数返回值：{result}")
        # 6. 返回原函数的返回值：确保装饰后的函数与原函数返回值一致，不破坏原逻辑
        return result
    # 7. 装饰器返回包装函数：用wrapper替换原函数的引用
    return wrapper

# 手动包装（不推荐，推荐用 @语法糖）
def add_original(a, b):
    return a + b
# 手动完成装饰器的绑定：传入原函数→获取包装函数→重新赋值
add_original = logger_decorator(add_original)
add_original(10, 20)
# 输出
# 调用函数：add_original，参数：(10, 20), {}
# 函数返回值：30
```

#### <2>@语法糖
```python

# 1. 定义装饰器函数（logger_decorator是装饰器，接收原函数作为参数）
def logger_decorator(func):  # func：接收被装饰的原函数（如后续的add函数）

    def wrapper(*args, **kwargs):
        print(f"调用函数：{func.__name__}，参数：{args}, {kwargs}")
        
        result = func(*args, **kwargs)
        
        print(f"函数返回值：{result}")
        
        return result
    return wrapper

# 使用@语法糖（简洁，推荐）
@logger_decorator  # 等价于：add = logger_decorator(add)
def add(a, b):  # 原函数：核心逻辑是计算两数之和，无额外功能
    return a + b
```
- 当程序执行到 **@装饰器名** 时，会自动完成 3 步操作：
	- 1.先定义下方的原函数（如 add）
	- 2.将原函数作为参数传入 @ 后面的装饰器函数（如 logger_decorator(add)）
	- 3.把装饰器函数返回的「包装函数」（如 wrapper）重新赋值给原函数名（如 add = wrapper）
	- 最终效果：后续调用原函数名（如 add(10,20)），实际调用的是装饰器返回的包装函数。
- **@语法的核心优势**：
	- **简洁性**：无需手动写“原函数 = 装饰器(原函数)”的赋值语句，一行 @装饰器名 即可完成装饰器绑定；
	- **可读性**：装饰器与原函数的关联关系直观，一眼就能看出“该函数被哪个装饰器增强”；
	- **可叠加性**：支持多个装饰器叠加使用（按从上到下的顺序依次装饰）
```python
@decorator1
@decorator2
def func():
等价于：func = decorator1(decorator2(func))
```

#### <3>@wraps的使用
- 在上述基础装饰器中，存在一个隐藏问题：
	- 装饰后的函数（如被装饰的add），其原始信息（如函数名__name__、文档字符串__doc__）会被包装函数wrapper覆盖，这会影响依赖函数原始信息的场景（如查看文档、反射等）
- **@wraps 的核心作用**：
	- 将原函数的元信息（函数名、文档字符串等）复制到包装函数中，保留原函数的身份信息。

##### 1）问题演示（未使用@wraps）
```python
def logger_decorator(func):
    def wrapper(*args, **kwargs):
        print(f"调用函数：{func.__name__}")
        result = func(*args, **kwargs)
        return result
    return wrapper

@logger_decorator
def add(a, b):
    """计算两个数的和"""
    return a + b

# 查看装饰后函数的信息（被wrapper覆盖）
print(f"函数名：{add.__name__}")  # 输出：wrapper（期望是add）
print(f"文档字符串：{add.__doc__}")  # 输出：None（期望是“计算两个数的和”）
```

##### 2）@wraps的使用方法（解决问题）
```python

from functools import wraps  # 必须导入wraps

def logger_decorator(func):
    @wraps(func)  # 关键：将func的元信息复制到wrapper
    def wrapper(*args, **kwargs):
        print(f"调用函数：{func.__name__}")
        result = func(*args, **kwargs)
        return result
    return wrapper

@logger_decorator
def add(a, b):
    """计算两个数的和"""
    return a + b

# 再次查看函数信息（已保留原函数信息）
print(f"函数名：{add.__name__}")  # 输出：add（正确）
print(f"文档字符串：{add.__doc__}")  # 输出：计算两个数的和（正确）
```

## 9.函数的实战技巧
### （1）文档字符串（规范写法）
技巧意义：提升代码可维护性与可读性，方便使用者快速了解函数信息，适配团队协作与规范化开发。
```python
def add(a, b):
    """
    计算两个数的和
    
    参数：
        a (int/float)：第一个数
        b (int/float)：第二个数
    
    返回值：
        int/float：两个数的和
    """
    return a + b

# 查看文档字符串
print(add.__doc__)
help(add)  # 更清晰的格式
```

### （2）函数注解
- **定义**：为参数/返回值添加类型提示（**仅提示，不强制校验**）
- **作用**：提高代码可读性，配合IDE自动提示，明确参数与返回值类型，降低传参错误，提升开发效率，支持静态类型校验。
```python
# 注解格式：参数: 类型，返回值 -> 类型
def add(a: int, b: int) -> int:
    return a + b

# 注解不影响执行（传入字符串也能运行）
print(add("hello", "world"))  # helloworld
# 查看注解
print(add.__annotations__)  # {'a': int, 'b': int, 'return': int}
```

### （3）偏函数（functools.partial）
- **定义**：
	- 偏函数是 Python functools 模块提供的工具，用于固定原函数的部分参数（位置参数或关键字参数），生成一个参数更少的新函数
	- 新函数调用时只需传入剩余未固定的参数，即可完成原函数的逻辑执行，核心价值是简化重复场景下的函数调用
```python
# 1. 导入偏函数工具（必须）
from functools import partial

# 2. 偏函数核心语法格式
新函数名 = partial(原函数名, 固定参数1=值1, 固定参数2=值2, ...)

# 语法说明：
# - 原函数名：需要被固定参数的原始函数
# - 固定参数：可指定原函数的任意位置参数（按顺序固定）或关键字参数（按参数名固定）
# - 新函数名：生成的简化版函数，调用时传入剩余参数即可
```
```python
from functools import partial

def add(a, b, c):
    return a + b + c

# 示例1：固定关键字参数c=10，生成新函数add_10
add_10 = partial(add, c=10)
print(add_10(1, 2))  # 13（1+2+10）

# 示例2：固定位置参数a=5，生成新函数add_5
add_5 = partial(add, 5)  # 按位置固定第一个参数a
print(add_5(2, 3))  # 10（5+2+3）
```

## 10.常用高阶辅助函数
高阶函数是指“接收函数作为参数”或“返回函数作为结果”的函数，是 Python 函数式编程的核心，常**与匿名函数（lambda）配合**，提升代码简洁性和可读性。

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260110231245247.png)

### （1）enumerate()：索引-元素对 迭代
- **定义**：
	- 遍历可迭代对象时，**同时返回元素的“索引”和“对应值”**，将两者打包为元组序列，返回迭代器（需用 list/tuple 转换查看）
- **语法**：
	- enumerate(可迭代对象, start=0)；

- **核心特点**：
	- 索引起始值默认 0，可通过 start 参数自定义（如 start=1 实现序号从 1 开始）；
	- 返回枚举迭代器，节省内存，遍历效率高于手动维护索引（如 i=0; for x in lst: ...; i+=1）；
	- 适用于需要同时获取“位置”和“元素值”的场景（如列表遍历标序号）。
```python
# 基础用法：默认索引从0开始
fruits = ["apple", "banana", "pear", "orange"]
for idx, fruit in enumerate(fruits):
    print(f"索引{idx}：{fruit}")  # 输出：索引0：apple；索引1：banana；索引2：pear；索引3：orange

# 进阶用法：自定义索引起始值（适合生成序号）
for seq, fruit in enumerate(fruits, start=1):
    print(f"序号{seq}：{fruit}")  # 输出：序号1：apple；序号2：banana；序号3：pear；序号4：orange

# 转换为列表查看完整结果
enum_result = list(enumerate(fruits))
print(enum_result)  # 输出：[(0, 'apple'), (1, 'banana'), (2, 'pear'), (3, 'orange')]
```

### （2）zip()：多序列元素打包
- **定义**：
	- 将多个可迭代对象（列表、元组等）的**“对应位置”**元素打包为**元组**，返回迭代器
	- 当多个序列长度不一致时，以最短的序列为准停止打包
- **语法**：
	- zip(可迭代对象1, 可迭代对象2, ...)
- **核心特点**：
	- 支持任意数量的可迭代对象（如 2 个、3 个甚至更多）
	- 返回 zip 迭代器，需用 list/tuple 转换查看具体元素
	- 可通过 \* 运算符解包 zip 结果，**内容**上能还原原始序列的元素，但**类型**上得到的是**元组**
		- **注意：返回的是“元组”！**
```python
# 基础用法1：打包两个序列（长度一致）并循环处理结果
names = ["张三", "李四", "王五"]
ages = [25, 30, 28]
user_info = zip(names, ages)

# 场景1：循环遍历打包结果，批量处理用户信息
for name, age in user_info:
    print(f"用户：{name}，年龄：{age}，是否成年：{age >= 18}")
# 输出：
# 用户：张三，年龄：25，是否成年：True
# 用户：李四，年龄：30，是否成年：True
# 用户：王五，年龄：28，是否成年：True

# 进阶用法1：打包多个序列（以最短为准）并转换为列表用于后续操作
scores = [85, 92]  # 长度比 names/ages 短
user_detail = zip(names, ages, scores)
detail_list = list(user_detail)  # 转换为列表便于多次使用（迭代器只能遍历一次）

# 场景2：根据打包数据生成用户成绩字典
score_dict = {name: (age, score) for name, age, score in detail_list}
print("用户成绩字典：", score_dict)
# 输出：用户成绩字典： {'张三': (25, 85), '李四': (30, 92)}

# 进阶用法2：解包zip结果（核心修改：元组转列表，支持修改）
packed = zip(names, ages)
unpacked_names, unpacked_ages = zip(*packed)  # 解包得到元组
# 关键步骤：将不可变的元组转为可变的列表
unpacked_names_list = list(unpacked_names)
unpacked_ages_list = list(unpacked_ages)

# 场景3：解包后批量更新序列数据（列表可直接修改）
# 方式1：列表推导式生成新列表（推荐）
updated_ages = [age + 1 for age in unpacked_ages_list]  # 所有人年龄+1
# 方式2：直接修改原列表（按需选择）
# unpacked_ages_list[0] = 100  # 直接修改张三的年龄为100

updated_user_info = zip(unpacked_names_list, updated_ages)
print("年龄更新后的数据：", list(updated_user_info))
# 输出：年龄更新后的数据： [('张三', 26), ('李四', 31), ('王五', 29)]
```

### （3）sorted()：通用排序
- **定义**：
	- 对任意可迭代对象（列表、元组、字符串等）**按指定规则排序**，返回“新的排序列表”，不会修改原可迭代对象
- **语法**：
	- sorted(可迭代对象, key=None, reverse=False)
- **核心特点**：
	- **key 参数**：
		- 接收函数作为参数，用于自定义排序规则（如按字符串长度、字典某字段排序）
	- **reverse 参数**：
		- 控制排序方向，默认 False（升序），True 为降序
	- **通用性强**：可用于所有可迭代对象（区别于列表的 sort() 方法，sort() 仅适用于列表且修改原列表）。
```python

# 基础用法：默认升序排序，后续用于计算排序后列表的总和
nums = [3, 1, 4, 1, 5, 9, 2]
sorted_nums = sorted(nums)
print(sorted_nums)  # 输出：[1, 1, 2, 3, 4, 5, 9]
print(nums)         # 输出：[3, 1, 4, 1, 5, 9, 2]（原列表未修改）
# 实际使用：计算排序后列表的总和
sorted_sum = sum(sorted_nums)
print(f"排序后列表的总和：{sorted_sum}")  # 输出：排序后列表的总和：25

# 进阶用法1：降序排序，后续用于筛选前3个最大值
desc_nums = sorted(nums, reverse=True)
print(desc_nums)    # 输出：[9, 5, 4, 3, 2, 1, 1]
# 实际使用：筛选前3个最大值并组成新列表
top3_nums = desc_nums[:3]
print(f"列表中的前3个最大值：{top3_nums}")  # 输出：列表中的前3个最大值：[9, 5, 4]

# 进阶用法2：自定义排序规则（按字符串长度排序），后续用于构建长度映射字典
strs = ["apple", "banana", "pear", "grape"]
sorted_strs = sorted(strs, key=len)  # key=len 表示按字符串长度排序
print(sorted_strs)  # 输出：['pear', 'apple', 'grape', 'banana']（长度：4→5→5→6）
# 实际使用：构建字符串与对应长度的映射字典
str_len_dict = {s: len(s) for s in sorted_strs}
print(f"字符串-长度映射字典：{str_len_dict}")  # 输出：字符串-长度映射字典：{'pear':4, 'apple':5, 'grape':5, 'banana':6}

# 进阶用法3：按字典字段排序，后续用于提取排序后的姓名列表
users = [{"name": "张三", "age": 25}, {"name": "李四", "age": 20}, {"name": "王五", "age": 30}]
sorted_users = sorted(users, key=lambda x: x["age"])  # 按 age 字段升序
print(sorted_users)  # 输出：[{'name': '李四', 'age': 20}, {'name': '张三', 'age': 25}, {'name': '王五', 'age': 30}]
# 实际使用：提取排序后的姓名，生成有序姓名列表
sorted_names = [user["name"] for user in sorted_users]
print(f"按年龄升序的姓名列表：{sorted_names}")  # 输出：按年龄升序的姓名列表：['李四', '张三', '王五']
```

### （4）max() / min()：最值获取
- **定义**：
	- max() 用于获取可迭代对象或多个零散参数的“最大值”
	- min() 用于获取“最小值”
- **语法**：
	- max(可迭代对象/参数组, key=None)
	- min(可迭代对象/参数组, key=None)
- **核心特点**：
	- 支持两种传参方式：
		- 直接传多个零散参数： 如max(2,5,1)
		- 传一个可迭代对象：如max([2,5,1])
	- **key 参数**：
		- 接收函数作为参数，自定义比较规则（如按字符串长度找最长/最短字符串）
	- **返回值**：
		- 单个最值（非迭代器），若可迭代对象为空则报错（需注意边界处理）。
```python
# 基础用法1：对可迭代对象取最值
nums = [3, 1, 4, 1, 5]
print(max(nums))  # 输出：5
print(min(nums))  # 输出：1

# 基础用法2：对多个零散参数取最值
print(max(2, 5, 1, 7))  # 输出：7
print(min(2, 5, 1, 7))  # 输出：1

# 进阶用法：自定义比较规则（按字符串长度找最值）
strs = ["apple", "banana", "pear"]
longest_str = max(strs, key=len)  # 最长字符串
shortest_str = min(strs, key=len)  # 最短字符串
print(longest_str)  # 输出：banana
print(shortest_str) # 输出：pear

# 进阶用法2：按字典字段找最值
users = [{"name": "张三", "score": 85}, {"name": "李四", "score": 92}, {"name": "王五", "score": 78}]
max_score_user = max(users, key=lambda x: x["score"])
min_score_user = min(users, key=lambda x: x["score"])
print(max_score_user)  # 输出：{'name': '李四', 'score': 92}
print(min_score_user)  # 输出：{'name': '王五', 'score': 78}
```

### （5）any() / all()：布尔值判断
- **定义**：
	- any() 和 all() 用于判断可迭代对象中元素的布尔值状态，返回单个布尔值；
	- **any()**：
		- 可迭代对象中**“任一元素为 True”则返回 True**，否则返回 False；
	- **all()**：
		- 可迭代对象中**“所有元素为 True”才返回 True**，否则返回 False。

- **语法**：
	- any(可迭代对象)；
	- all(可迭代对象)；

- **核心特点**：
	- 短路求值：
		- any() 找到第一个 True 就停止遍历，all() 找到第一个 False 就停止遍历，效率高；
	- 空可迭代对象**特殊处理**：
		- any(空序列) 返回 False，**all(空序列) 返回 True**（逻辑上理解为"不存在 False 的元素"，即"没有反例"）
	- 仅关注布尔值：
		- 不处理元素本身，仅判断元素的“真值”（0、空字符串、空列表等为 False，非空/非0为 True）。
```python
# 基础用法：any() 判断“任一为真”
list1 = [False, 0, "", "hello"]  # 存在 "hello"（True）
list2 = [False, 0, ""]           # 所有元素均为 False
print(any(list1))  # 输出：True
print(any(list2))  # 输出：False

# 基础用法：all() 判断“所有为真”
list3 = [True, 1, "hello"]  # 所有元素均为 True
list4 = [True, 1, ""]       # 存在 ""（False）
print(all(list3))  # 输出：True
print(all(list4))  # 输出：False

# 进阶用法：空序列处理
empty_list = []
print(any(empty_list))  # 输出：False
print(all(empty_list))  # 输出：True

# 实战场景：判断列表是否存在负数
nums = [3, -2, 5, 7]
has_negative = any(num < 0 for num in nums)
print(has_negative)  # 输出：True

# 实战场景：判断列表所有元素均为正数
all_positive = all(num > 0 for num in nums)
print(all_positive)  # 输出：False
```

### （6）filter()：元素筛选
- **定义**：
	- 接收一个“筛选函数”和一个可迭代对象，遍历可迭代对象的每个元素，**保留“筛选函数返回 True”的元素**，返回迭代器（需用 list/tuple 转换查看结果）；
- **语法**：
	- filter(筛选函数, 可迭代对象)；
- **核心特点**：
	- 筛选函数可为 None：此时直接判断元素自身的布尔值（保留 True 的元素）；
	- 返回 filter 迭代器，节省内存，仅在遍历/转换时才执行筛选；
	- 仅保留符合条件的“原始元素”，不修改元素本身。
```python
# 基础用法1：自定义筛选函数（筛选偶数）
nums = [1, 2, 3, 4, 5, 6, 7, 8]
def is_even(num):
    return num % 2 == 0  # 偶数返回 True，奇数返回 False

even_nums = filter(is_even, nums)
even_list = list(even_nums)  # 转换为列表便于后续使用
# 使用筛选结果进行后续计算（求偶数和）
even_sum = sum(even_list)
print(f"筛选出的偶数列表：{even_list}")  # 输出：[2, 4, 6, 8]
print(f"偶数列表的和：{even_sum}")  # 输出：20

# 基础用法2：筛选函数为 None（保留非空元素）
strs = ["", "张三", "", "李四", "王五", ""]
non_empty_strs = filter(None, strs)
non_empty_list = list(non_empty_strs)
# 使用筛选结果构建字典（姓名-序号映射）
name_dict = {name: idx+1 for idx, name in enumerate(non_empty_list)}
print(f"非空姓名列表：{non_empty_list}")  # 输出：['张三', '李四', '王五']
print(f"姓名序号字典：{name_dict}")  # 输出：{'张三': 1, '李四': 2, '王五': 3}

# 进阶用法：结合 lambda 简化筛选（筛选年龄大于25的用户）
users = [("张三", 25), ("李四", 30), ("王五", 28), ("赵六", 22)]
adult_users = filter(lambda x: x[1] > 25, users)
adult_list = list(adult_users)
# 使用筛选结果提取姓名，生成成年用户名单
adult_names = [user[0] for user in adult_list]
print(f"成年用户列表：{adult_list}")  # 输出：[('李四', 30), ('王五', 28)]
print(f"成年用户名单：{adult_names}")  # 输出：['李四', '王五']
```

### （7）map()：元素映射转换
- **定义**：
	- 接收一个“映射函数”和一个/多个可迭代对象，对可迭代对象的**每个元素“应用”映射函数**，将函数执行结果作为新元素，返回迭代器（需用 list/tuple 转换查看）
- **语法**：
	- map(映射函数, 可迭代对象1, 可迭代对象2, ...)
- **核心特点**：
	- 映射函数可为普通函数、匿名函数（lambda），也可为内置函数（如 int、str）
	- 支持多个可迭代对象：映射函数需接收对应数量的参数，以最短的可迭代对象为准停止；
	- 返回 map 迭代器，元素为映射函数的执行结果（实现元素的批量转换）。
```python

# 基础用法1：单个可迭代对象（元素批量乘2，后续用于求和计算）
nums = [1, 2, 3, 4, 5]
double_nums = map(lambda x: x * 2, nums)
double_list = list(double_nums)  # 转换为列表便于后续使用
# 实际使用：计算翻倍后的总和
double_sum = sum(double_list)
print(f"元素翻倍列表：{double_list}")  # 输出：[2, 4, 6, 8, 10]
print(f"翻倍后总和：{double_sum}")  # 输出：30

# 基础用法2：多个可迭代对象（两列表元素对应相加，后续用于筛选大于30的结果）
list1 = [10, 20, 30, 40]
list2 = [5, 15, 25]  # 长度较短，以该列表为准停止
sum_list = map(lambda x, y: x + y, list1, list2)
sum_result = list(sum_list)
# 实际使用：筛选相加结果大于30的数据
filtered_sum = [num for num in sum_result if num > 30]
print(f"两列表对应相加结果：{sum_result}")  # 输出：[15, 35, 55]
print(f"相加结果中大于30的数：{filtered_sum}")  # 输出：[35, 55]

# 进阶用法：内置函数作为映射函数（字符串转整数，后续用于求平均值）
str_nums = ["1", "2", "3", "4", "5"]
int_nums = map(int, str_nums)
int_list = list(int_nums)
# 实际使用：计算转换后的整数平均值
average = sum(int_list) / len(int_list)
print(f"字符串转整数列表：{int_list}")  # 输出：[1, 2, 3, 4, 5]
print(f"整数列表的平均值：{average}")  # 输出：3.0

# 进阶用法：自定义函数实现多元素转换（格式化用户信息，后续用于生成用户列表）
def format_user(name, age):
    return f"姓名：{name}，年龄：{age}"

names = ["张三", "李四", "王五"]
ages = [25, 30, 28]
formatted_users = map(format_user, names, ages)
formatted_list = list(formatted_users)
# 实际使用：将格式化后的用户信息写入模拟文件
with open("user_info.txt", "w", encoding="utf-8") as f:
    for user in formatted_list:
        f.write(user + "\n")
print(f"格式化用户信息列表：{formatted_list}")  # 输出：['姓名：张三，年龄：25', '姓名：李四，年龄：30', '姓名：王五，年龄：28']
print("用户信息已写入 user_info.txt 文件")
```

### （8）reduce()：累积计算
- **定义**：
	- 接收一个“累积函数”和一个可迭代对象，从左到右遍历可迭代对象，将前两个元素的计算结果与下一个元素继续累积，最终返回“单个累积结果”（非迭代器）；
- **语法**：
	- reduce(func, iterable[, initial])
	- 需从 functools 模块导入
- **核心特点**：
	- 必须导入：
		- reduce() 不在 Python 内置函数中，需通过 **from functools import reduce** 导入；
	- initial 参数（可选）：
		- 为累积计算设置初始值（初始值将参与第一次计算，如 initial=10 时，先计算 10+第一个元素）；
	- 累积函数要求：
		- 必须接收两个参数（前一次累积结果和当前元素），返回新的累积结果。
```python
from functools import reduce

# 基础用法1：累积求和
nums = [1, 2, 3, 4, 5]
sum_result = reduce(lambda x, y: x + y, nums)
print(sum_result)  # 输出：15（计算过程：((((1+2)+3)+4)+5)）

# 基础用法2：累积求积
product_result = reduce(lambda x, y: x * y, nums)
print(product_result)  # 输出：120（计算过程：((((1*2)*3)*4)*5)）

# 进阶用法：设置初始值（初始值参与第一次计算）
sum_with_init = reduce(lambda x, y: x + y, nums, 10)  # 初始值10
print(sum_with_init)  # 输出：25（计算过程：(((((10+1)+2)+3)+4)+5)）

# 进阶用法：自定义累积函数（拼接字符串）
strs = ["Hello", " ", "Python", " ", "World"]
def concat_str(x, y):
    return x + y

final_str = reduce(concat_str, strs)
print(final_str)  # 输出：Hello Python World
```

# 九、类

## 1.类的基础概念和定义
### （1）核心概念
- **类（class）**：是对一类具有相同属性和方法的对象的抽象描述，相当于一个“模板”。
	- 例如，“动物”类可以抽象出所有动物共有的属性（如名字、年龄）和方法（如进食、移动）。
- **对象（object）**：是类的实例化结果，是具体的实体。
	- 例如，“一只名叫旺财的狗”是“动物”类的一个对象，具备具体的名字（旺财）和年龄（3岁）。
- **面向对象三大特征**：封装、继承、多态，是类设计的核心思想。

### （2）类的定义语法
使用 **class** 关键字定义类，基本语法如下：
```python
# 基础类定义
class 类名(父类列表):  # 父类列表可选，默认继承object（Python3中所有类的基类）
    """类的文档字符串（用于说明类的功能）"""
    # 类属性（属于类本身，所有实例共享）
    类属性名 = 属性值
    
    # 构造方法（初始化实例属性）
    def __init__(self, 参数列表):
        # 实例属性（属于单个实例，每个实例独立）
        self.实例属性名 = 参数值
    
    # 类的方法（用于操作属性、实现功能）
    def 方法名(self, 参数列表):
        方法体
        return 返回值

# 实例化对象（创建类的具体实体）
对象名 = 类名(参数列表)  # 参数列表对应__init__方法的参数（除self外）
```
```python
# 定义Animal类
class Animal:
    """动物类，封装动物的基本属性和行为"""
    # 类属性：所有动物共有的属性（如所属界别）
    kingdom = "动物界"
    
    # 构造方法：初始化实例属性（名字、年龄）
    def __init__(self, name, age):
        # self：必须作为第一个参数，指代当前实例本身
        self.name = name  # 实例属性：名字
        self.age = age    # 实例属性：年龄
    
    # 实例方法：动物的行为（进食）
    def eat(self, food):
        print(f"{self.name}（{self.age}岁）正在吃{food}")
    
    # 实例方法：动物的行为（移动）
    def move(self):
        print(f"{self.name}正在移动")

# 1. 实例化对象（创建具体的动物）
dog = Animal("旺财", 3)  # dog是Animal类的一个实例
cat = Animal("咪宝", 2)  # cat是另一个实例

# 2. 访问类属性（所有实例共享）
print(Animal.kingdom)  # 输出：动物界
print(dog.kingdom)     # 输出：动物界（实例可访问类属性）
print(cat.kingdom)     # 输出：动物界

# 3. 访问实例属性（每个实例独立）
print(dog.name)  # 输出：旺财
print(cat.name)  # 输出：咪宝

# 4. 调用实例方法
dog.eat("骨头")  # 输出：旺财（3岁）正在吃骨头
cat.move()       # 输出：咪宝正在移动
```

### （3）\_\_init\_\_：核心构造方法
\_\_init\_\_是Python类中最核心的魔术方法（特殊方法）之一，专门用于**在实例化对象时完成实例属性的初始化**，相当于对象的“出生初始化器”。它是连接类模板与具体对象的关键桥梁，决定了对象创建时必须具备的初始状态。
#### <1>核心特征
- **自动触发执行**：
	- 仅在通过 **类名()** 进行实例化时**自动调用**，无需手动调用。
	- 例如：执行dog = Animal("旺财", 3)时，Python会自动触发Animal.__init__(dog, "旺财", 3)的执行。
- **强制首个参数为self**
	- **self**是Python的内置约定，用于**指代**当前正在创建的**实例对象本身**。
	- Python会自动将新创建的实例作为参数传递给self，开发者无需手动传入。
- **无默认返回值**：
	- \_\_init\_\_方法的**返回值必须是None**，不能返回任何其他数据类型（否则会直接抛出TypeError异常）。
- **实例属性的载体**：
	- **通过self.属性名 = 值的语法，将数据绑定到当前实例上**，这些属性会成为实例的**专属数据**，实现不同实例间的**数据隔离**。
```python
class 类名:
    def __init__(self, 参数1, 参数2, ...):
        # 初始化实例属性
        self.属性1 = 参数1
        self.属性2 = 参数2
        # 可定义默认属性（无需外部传入）
        self.属性3 = 默认值

# 实例化时传入参数（匹配__init__除self外的参数）
对象 = 类名(参数1, 参数2, ...)
```
```python
# 定义Person类，通过__init__初始化姓名和年龄属性
class Person:
    def __init__(self, name, age):
        # 将外部传入的name绑定为实例属性self.name
        self.name = name
        # 将外部传入的age绑定为实例属性self.age
        self.age = age
        # 定义默认实例属性（无需外部传入）
        self.gender = "未知"

# 实例化对象：传入name和age参数，__init__自动执行
p1 = Person("张三", 18)
p2 = Person("李四", 20)

# 访问实例属性（每个实例的属性独立）
print(p1.name, p1.age, p1.gender)  # 输出：张三 18 未知
print(p2.name, p2.age, p2.gender)  # 输出：李四 20 未知
```

#### <2>\_\_init\_\_方法的高级用法
##### 1）参数默认值：提升初始化灵活性
可以为__init__的参数设置默认值，实例化时可省略这些参数，使用默认值初始化属性。
```python
class Student:
    def __init__(self, name, age=18, score=60):
        # age默认值18，score默认值60
        self.name = name
        self.age = age
        self.score = score

# 实例化时传入不同数量的参数
s1 = Student("王五")  # 仅传name，age和score用默认值
s2 = Student("赵六", 19)  # 传name和age，score用默认值
s3 = Student("孙七", 20, 95)  # 传所有参数

print(s1.age, s1.score)  # 输出：18 60
print(s2.age, s2.score)  # 输出：19 60
print(s3.age, s3.score)  # 输出：20 95
```

##### 2）可变参数：支持动态传入多个属性
通过\*args（可变位置参数）和\*\*kwargs（可变关键字参数），让\_\_init\_\_支持动态传入任意数量的参数，适配更复杂的初始化场景。
```python
class Product:
    def __init__(self, name, price, *args, **kwargs):
        self.name = name
        self.price = price
        # 处理可变位置参数（如商品标签）
        self.tags = args
        # 处理可变关键字参数（如产地、保质期等额外属性）
        for key, value in kwargs.items():
            setattr(self, key, value)

# 实例化时传入固定参数、可变位置参数、可变关键字参数
p = Product("手机", 2999, "电子产品", "智能", origin="中国", shelf_life="2年")

print(p.name, p.price)  # 输出：手机 2999
print(p.tags)  # 输出：('电子产品', '智能')
print(p.origin, p.shelf_life)  # 输出：中国 2年
```
**补充**：
- setattr() 是 Python 内置函数，作用是**动态**给对象（类实例 / 类本身）设置属性
- 语法： setattr(对象, 属性名, 属性值)
	- 等价于 **对象.属性名 = 属性值**，但支持动态传入属性名字符串（灵活度更高）
	- **属性不存在则创建，存在则覆盖原有值** 


## 2.类的属性
属性分为 **「类属性」** 和 **「实例属性」** ，核心区别在于**作用域（是否共享）**。

### （1）类属性
- **定义位置**：在类体中、所有方法之外。
- **作用域**：属于类本身，所有实例**共享**该属性，修改类属性会影响所有实例。
- **访问方式**：
	- **类名.类属性名（推荐）**
	- 实例名.类属性名（不推荐，易混淆实例属性）
```python
class Student:
    # 类属性：所有学生共享的学校名称
    school = "北京大学"
    
    def __init__(self, name):
        self.name = name  # 实例属性：学生姓名

# 访问类属性
print(Student.school)  # 输出：北京大学

# 多个实例共享类属性
stu1 = Student("张三")
stu2 = Student("李四")
print(stu1.school)  # 输出：北京大学
print(stu2.school)  # 输出：北京大学

# 修改类属性（影响所有实例）
Student.school = "清华大学"
print(stu1.school)  # 输出：清华大学
print(stu2.school)  # 输出：清华大学

# 注意：实例不能直接修改类属性（会创建同名实例属性覆盖类属性）
stu1.school = "复旦大学"
print(stu1.school)  # 输出：复旦大学（实例属性覆盖）
print(stu2.school)  # 输出：清华大学（类属性未变）
print(Student.school)  # 输出：清华大学（类属性未变）

```

#### 补充：类属性必须在类体中定义时就赋值（哪怕赋 None）
- **原因**：类属性的核心是 “赋值”，仅写属性名没有赋值，Python 不会将其注册为类的属性，自然无法访问。
```python
# 1.错误示例
class Student:
	school  # ❌ 空声明，不是有效类属性（无赋值）

# 2.正确示例
class Student:
	school="北京大学"

# 3.类属性的 “延迟赋值”（特殊场景）
class Student:
    # 先声明类属性（无初始值），后续赋值
    school = None  # 先赋空值，避免空声明
    
# 后续给类属性赋值（仍无需__init__）
Student.school = "复旦大学"

print(Student.school)  # 输出：复旦大学
```


### （2）实例属性
- **定义位置**：
	- 通常**在 __init__ 方法中**
	- 通过 **self.属性名** 定义。
- **作用域**：
	- 属于单个实例，**每个实例的属性独立**
	- 修改一个实例的属性不影响其他实例。
- **访问方式**：
	- **实例名.实例属性名**
```python
class Person:
    def __init__(self, name, height):
        self.name = name      # 实例属性：姓名
        self.height = height  # 实例属性：身高
    
    # 方法中访问实例属性
    def show_info(self):
        print(f"姓名：{self.name}，身高：{self.height}cm")

# 实例1
p1 = Person("张三", 180)
# 实例2
p2 = Person("李四", 175)

# 访问实例属性
p1.show_info()  # 输出：姓名：张三，身高：180cm
p2.show_info()  # 输出：姓名：李四，身高：175cm

# 修改实例属性（仅影响当前实例）
p1.height = 182
p1.show_info()  # 输出：姓名：张三，身高：182cm
p2.show_info()  # 输出：姓名：李四，身高：175cm（无变化）
```

**补充**
- 实例属性**推荐**在 __init__ 中通过 self.属性 = 值 初始化（确保所有实例具有一致的属性结构）
- 实例属性也可以在其他方法中动态添加，但**不推荐**（会导致不同实例属性不一致）
- 实例属性也可以是类，即**组合**，详情见[类的组合](#class-composition)

### （3）私有属性（隐藏属性）
Python中没有严格的“私有”关键字，通过属性名前缀的下划线区分不同访问权限的属性，主要分为**单下划线 \_**属性和**双下划线 \_\_**属性两类，核心作用是控制属性的访问范围，体现封装特性。

#### <1>单下划线属性（\_attr）
单下划线属性是Python社区的**约定俗成规范**，并非语法级别的限制，用于标识“不建议外部直接访问”的属性，但**外部仍可直接访问和修改**，仅作为开发者之间的沟通信号（类似“内部使用，谨慎操作”的提醒）。
- 定义：self.\_保护属性名 = 属性值。
- 访问：外部可直接访问（实例名.\_保护属性名），无语法限制。
- 用途：标记类的内部属性，提示外部尽量通过类提供的公开方法操作，而非直接修改。
```python
# 单下划线保护属性示例
class Person:
    def __init__(self, name, age):
        self.name = name          # 公开属性
        self._age = age           # 保护属性（约定不建议外部直接访问）
    
    # 公开方法操作保护属性
    def get_age(self):
        return self._age
    
    def set_age(self, new_age):
        if new_age > 0 and new_age < 150:
            self._age = new_age
        else:
            print("年龄输入无效！")

p = Person("张三", 18)
# 外部可直接访问保护属性（不推荐）
print(p._age)  # 输出：18
# 外部可直接修改保护属性（不推荐）
p._age = 20
print(p._age)  # 输出：20
# 推荐：通过公开方法操作
p.set_age(25)
print(p.get_age())  # 输出：25
```

#### <2>双下划线属性（\_\_attr）
双下划线属性是Python的语法特性，通过**名称修饰（Name Mangling）机制**实现真正的访问限制，外部无法直接通过原属性名访问，只能通过类内部的方法操作。
##### 1）核心原理：名称修饰（Name Mangling）
- 当类中定义双下划线属性时，Python解释器会自动将属性名修改为 **\_类名\_\_属性名** 的格式（即 \_ClassName\_\_attr），从而隐藏原属性名，避免外部直接访问。
- 这种修改是语法层面的，并非加密，**外部仍可通过修改后的名称访问**（不推荐，违反封装设计）。

##### 2）定义与访问规则：
- 定义：self.\_\_私有属性名 = 属性值。
- 访问限制：外部直接通过 实例名.\_\_私有属性名 会报错（AttributeError）。
- 合法访问：只能通过类内部的“getter/setter”等公开方法访问和修改。
- 特殊访问（不推荐）：外部可通过 **实例名.\_类名\_\_私有属性名** 访问修饰后的属性。
```python
# 双下划线属性原理解析
class User:
    def __init__(self, username, password):
        self.username = username          # 公开属性
        self.__password = password        # 双下划线私有属性（会被名称修饰）
    
    # getter方法：类内部访问私有属性（正常）
    def get_password(self):
        # 类内部可直接使用原属性名访问
        return self.__password
    
    # setter方法：类内部修改私有属性（正常）
    def set_password(self, new_password):
        if len(new_password) >= 6:
            self.__password = new_password
        else:
            print("密码长度不能小于6位！")

# 实例化
user = User("zhangsan", "123456")

# 1. 访问公开属性（正常）
print(user.username)  # 输出：zhangsan

# 2. 直接访问原私有属性名（报错：名称已被修饰）
# print(user.__password)  # AttributeError: 'User' object has no attribute '__password'

# 3. 查看实例的属性列表（可看到修饰后的属性名）
print(user.__dict__)  # 输出：{'username': 'zhangsan', '_User__password': '123456'}

# 4. 外部通过修饰后的名称访问（不推荐，破坏封装）
print(user._User__password)  # 输出：123456
# 外部通过修饰后的名称修改（不推荐）
user._User__password = "abc123456"
print(user.get_password())  # 输出：abc123456

# 5. 通过合法的getter/setter方法操作（推荐）
print(user.get_password())  # 输出：abc123456
user.set_password("654321")
print(user.get_password())  # 输出：654321
```

### （4）补充： 实例属性名写错会 “静默创建新属性” 的问题
#### <1>问题描述
**问题本质**：Python 是动态类型语言，属性 / 变量无需提前声明，不会在运行前检查属性 / 变量是否存在
- 若你手滑写成 stu1.scholl = "复旦大学"（多写一个 l），Python 不会报错，而是默默创建一个全新的 scholl 实例属性，原 school 类属性 / 实例属性完全不受影响。
- 这种错误在运行前（静态检查）很难发现，只有运行时用到该属性时才会暴露。
```python
class Student:
    school = "北京大学"
    def __init__(self, name):
        self.name = name

stu1 = Student("张三")
# 手滑把 school 写成 scholl（拼写错误）
stu1.scholl = "复旦大学"  

# 此时访问正确属性名，还是原来的类属性值，你可能误以为修改成功
print(stu1.school)  # 输出：北京大学（完全没达到修改预期）
# 而错误的属性名被悄悄创建，你可能完全没意识到
print(stu1.scholl)  # 输出：复旦大学（这个属性本不该存在）
```

#### <2>常用应对方法
##### 1）使用 \_\_slots\_\_（最直接）
通过在类中定义 \_\_slots\_\_，可以指定实例**只能拥有“实例属性”**，一旦尝试创建未声明的“实例属性”，会直接抛出 AttributeError，从根源上阻止拼写错误导致的 “静默创建新实例属性”：
```python
class Student:
    # 限制实例只能有 name/school 两个属性，其他实例属性创建会报错
    __slots__ = ["name"]  
    school = "北京大学"
    
    def __init__(self, name):
        self.name = name

stu1 = Student("张三")
# 拼写错误：scholl 不在 __slots__ 中，直接报错
stu1.scholl = "复旦大学"  
# 报错：AttributeError: 'Student' object has no attribute 'scholl'
```
##### 补充：\_\_slots\_\_ 的原理 与 继承规则
1. **动态添加实例属性的本质**：
   - 默认情况下，实例通过 \_\_dict\_\_ 字典存储属性，因此可以动态添加任意属性

2. **slots的原理**：
- 为实例指定**固定的属性名列表**，用固定内存位置替代 \_\_dict\_\_
- 实例只能拥有 \_\_slots\_\_ 中声明的属性

3. **slots 的继承规则**
- **规则 1**：子类不自动继承父类的 \_\_slots\_\_
     - 父类有 \_\_slots\_\_、子类无 → 子类实例有 \_\_dict\_\_，可自由添加属性
- **规则 2**：子类会继承父类的 \_\_dict\_\_（如果有）
     - 父类无 \_\_slots\_\_、子类有 → 子类的 \_\_slots\_\_ 被绕过
- **规则 3**：父子都有 \_\_slots\_\_ → 实例可用属性为两者叠加

4. **判断关键**：
   - 看实例最终是否有 \_\_dict\_\_ → 有则可自由添加，无则受限于 \_\_slots__
   - \_\_slots\_\_ 中包含 '\_\_dict\_\_' 时，相当于恢复动态能力

```python
# 规则 1：子类不自动继承父类的 __slots__
class Parent:
    __slots__ = ['name']

class Child(Parent):
    pass  # 子类未定义 __slots__

c = Child()
c.name = "张三"      # ✅ 继承自父类 slots
c.age = 18           # ✅ 子类有 __dict__，可自由添加
print(hasattr(c, '__dict__'))  # True

# 子类继承父类的 __dict__，导致子类 __slots__ 无效
class Parent:
    pass  # 无 __slots__，有 __dict__

class Child(Parent):
    __slots__ = ['name']

c = Child()
c.name = "张三"      # ✅
c.age = 18           # ✅ 不报错！因为继承了 __dict__
print(hasattr(c, '__dict__'))  # True
print(c.__dict__)    # {'age': 18}

# 规则 3：父子都有 __slots__，叠加生效
class Parent:
    __slots__ = ['name']

class Child(Parent):
    __slots__ = ['age']

c = Child()
c.name = "张三"      # ✅ 来自父类 slots
c.age = 18           # ✅ 来自子类 slots
print(hasattr(c, '__dict__'))  # False

# 尝试添加 slots 外的属性
try:
    c.gender = "男"
except AttributeError as e:
    print(e)  # 'Child' object has no attribute 'gender'

# 补充1：__slots__ 中包含 '__dict__'
class Parent:
    __slots__ = ['name']

class Child(Parent):
    __slots__ = ['age', '__dict__']

c = Child()
c.name = "张三"
c.age = 18
c.gender = "男"      # ✅ 可以，因为有 __dict__
print(hasattr(c, '__dict__'))  # True
print(c.__dict__)    # {'gender': '男'}

# 补充2：空 __slots__ 禁用一切动态属性
class Empty:
    __slots__ = []

e = Empty()
try:
    e.name = "张三"
except AttributeError as e:
    print(e)  # 'Empty' object has no attribute 'name'
```

##### 3）封装修改属性的方法（而非直接赋值）
通过自定义方法修改属性，在方法内校验属性名 / 值的合法性，避免直接赋值的随意性：
```python
class Student:
    school = "北京大学"
    def __init__(self, name):
        self.name = name
    
    def set_school(self, new_school):
        # 方法内只处理正确的属性名，避免拼写错误
        self.school = new_school

stu1 = Student("张三")
# 只能通过方法修改，无法写错属性名（方法名错会直接报错，更容易发现）
stu1.set_school("复旦大学")
print(stu1.school)  # 输出：复旦大学（正确修改）
# 若手滑写 stu1.set_scholl("复旦大学")，会直接报 AttributeError，立刻发现错误
```

##### 3）IDE 提示
PyCharm/VSCode 等DIE中会给出**“警告提示”**，看见时需注意


## 3.类的方法
方法是类中定义的函数，根据参数和作用域可分为 **「实例方法」** 、 **「类方法」**、 **「静态方法」** 三类。

### （1）实例方法
- **定义**：
	- **第一个参数必须是 self**（指代当前实例）
	- 通过 self 访问实例属性和其他实例方法。
- **调用**：
	- 只能通过「实例」调用（**实例名.方法名()**），self 由Python自动传递，无需手动传入。
- **用途**：
	- 操作实例属性，实现与具体实例相关的功能。
```python
# 定义包含实例方法的类
class Person:
    # 构造方法初始化实例属性
    def __init__(self, name, age):
        self.name = name  # 实例属性：姓名
        self.age = age    # 实例属性：年龄
    
    # 实例方法1：访问实例属性
    def show_info(self):
        # 通过self访问当前实例的属性
        print(f"姓名：{self.name}，年龄：{self.age}岁")
    
    # 实例方法2：修改实例属性
    def grow_up(self, years=1):
        # 通过self修改实例属性
        self.age += years
        print(f"{self.name}长大了{years}岁，现在{self.age}岁")
        # 通过self调用其他实例方法
        self.show_info()

# 1. 实例化对象（触发__init__初始化属性）
p = Person("张三", 18)

# 2. 调用实例方法（无需传递self，Python自动绑定）
p.show_info()  # 输出：姓名：张三，年龄：18岁

# 3. 调用带参数的实例方法
p.grow_up(2)  # 输出：张三长大了2岁，现在20岁；姓名：张三，年龄：20岁

# 4. 再次调用实例方法，验证属性已修改
p.show_info()  # 输出：姓名：张三，年龄：20岁
```

### （2）类方法
使用 **@classmethod 装饰器定义**，**用于操作类属性**，不依赖实例。
- **定义**：
	- 第一个参数必须是 cls（**指代当前类**）
	- 通过 cls 访问类属性和其他类方法。
- **调用**：
	- **可通过「类」或「实例」调用**（推荐用类调用：类名.方法名()），cls 由Python自动传递。
- **用途**：
	- 修改类属性、创建实例（替代构造方法）等与类相关的操作。
```python
class Car:
    # 类属性：汽车总数
    total_cars = 0
    
    def __init__(self, brand):
        self.brand = brand  # 实例属性：品牌
        Car.total_cars += 1  # 每创建一个实例，汽车总数+1
    
    # 类方法：获取汽车总数（操作类属性）
    @classmethod
    def get_total_cars(cls):
        print(f"当前汽车总数：{cls.total_cars}")
        return cls.total_cars
    
    # 类方法：创建默认品牌的汽车（替代构造方法）
    @classmethod
    def create_default_car(cls):
        return cls(brand="丰田")  # 等价于 Car("丰田")

# 1. 实例化汽车
car1 = Car("宝马")
car2 = Car("奔驰")

# 2. 调用类方法（类调用）
Car.get_total_cars()  # 输出：当前汽车总数：2

# 3. 实例调用类方法（也可，但不推荐）
car1.get_total_cars()  # 输出：当前汽车总数：2

# 4. 通过类方法创建实例
default_car = Car.create_default_car()
print(default_car.brand)  # 输出：丰田
Car.get_total_cars()  # 输出：当前汽车总数：3
```

### （3）静态方法
使用 **@staticmethod 装饰器**定义，既不依赖实例（无 self），也不依赖类（无 cls），相当于类中的“普通函数”。
- **定义**：
	- 无强制第一个参数self，参数自由定义。
- **调用**：
	- 可通过「类」或「实例」调用（推荐用类调用）。
- **用途**：
	- 实现与类和实例都无关的辅助功能（如工具函数）。
```python
class MathTools:
    """数学工具类，包含静态方法实现辅助计算"""
    
    # 静态方法：计算两个数的平方和
    @staticmethod
    def sum_of_squares(a, b):
        return a**2 + b**2
    
    # 静态方法：判断一个数是否为偶数
    @staticmethod
    def is_even(num):
        return num % 2 == 0

# 1. 类调用静态方法（推荐）
print(MathTools.sum_of_squares(3, 4))  # 输出：25（3²+4²=25）
print(MathTools.is_even(6))            # 输出：True

# 2. 实例调用静态方法（也可）
tools = MathTools()
print(tools.sum_of_squares(5, 12))  # 输出：169（5²+12²=169）
print(tools.is_even(7))             # 输出：False
```

### （4）方法对比总结
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111104147036.png)

## 4.面向对象三大特性：封装、继承、多态
### （1）封装
- **核心思想**：
	- 将类的属性和方法“包裹”起来，**隐藏内部实现细节，只对外提供有限的访问接口（如getter/setter方法），防止外部随意修改内部数据**，保证数据的安全性和代码的可维护性。
- **实现方式**：
	- 使用 \_\_ 定义私有属性/私有方法（外部无法直接访问）。
	- 提供公开的getter（获取）、setter（修改）方法，控制对私有属性的访问逻辑。
```python
class BankAccount:
    """银行账户类，封装账户余额的访问和修改"""
    def __init__(self, account_id, balance=0):
        self.account_id = account_id      # 公开属性：账号
        self.__balance = balance          # 私有属性：余额（隐藏）
    
    # 私有方法：日志记录（内部使用，外部无法调用）
    def __log(self, action, amount):
        print(f"账号{self.account_id}：{action} {amount}元，当前余额：{self.__balance}元")
    
    # getter方法：查询余额
    def get_balance(self):
        self.__log("查询余额", 0)
        return self.__balance
    
    # setter方法：存款（控制存款逻辑）
    def deposit(self, amount):
        if amount > 0:
            self.__balance += amount
            self.__log("存款", amount)
        else:
            print("存款金额必须大于0！")
    
    # setter方法：取款（控制取款逻辑）
    def withdraw(self, amount):
        if amount > 0 and amount <= self.__balance:
            self.__balance -= amount
            self.__log("取款", amount)
            return amount
        else:
            print("取款金额无效或余额不足！")
            return 0

# 实例化账户
account = BankAccount("6222081234567890", 1000)

# 访问公开属性
print(account.account_id)  # 输出：6222081234567890

# 通过公开方法操作私有属性
account.deposit(500)        # 输出：账号6222081234567890：存款 500元，当前余额：1500元
account.withdraw(300)       # 输出：账号6222081234567890：取款 300元，当前余额：1200元
print(account.get_balance())# 输出：账号6222081234567890：查询余额 0元，当前余额：1200元

# 尝试直接访问私有属性/方法（报错）
# print(account.__balance)  # AttributeError
# account.__log("测试", 0)  # AttributeError

```

### （2）继承（Inheritance）
#### <1>介绍
- **核心思想**：
	- **子类（派生类）继承父类（基类）的属性和方法**，无需重复编写代码，同时子类可扩展新的属性和方法，或重写父类的方法（覆盖父类逻辑），实现代码复用和扩展。
	- Python支持「单继承」和「多继承」（一个子类继承多个父类），但多继承易导致逻辑复杂，通常推荐单继承。
- **基本语法**：
```python
# 父类（基类）
class 父类名:
    父类属性和方法

# 子类（派生类）继承父类
class 子类名(父类名):
    # 子类可扩展新的属性和方法
    子类属性和方法
    
    # （可选）重写父类方法（覆盖父类逻辑）
    def 父类方法名(self, 参数列表):
        子类的方法体

```
- **关键概念**
	- **继承传递性**：
		- 如果B继承A，C继承B，则C同时拥有A和B的属性和方法。
	- **重写（Override）**：
		- 子类定义与父类同名的方法，覆盖父类的实现逻辑（当调用该方法时，优先执行子类逻辑）。
	- **super()**：
		- 在子类中调用父类的方法（如 **super().\_\_init\_\_()** 调用父类的构造方法）。

**注意！！**：如果要继承父类的属性，**子类\_\_init\_\_必须显式调用父类\_\_init\_\_**
- 子类自定义了\_\_init\_\_方法，会覆盖父类的\_\_init\_\_方法。
- 若子类需要继承父类的实例属性，必须通过 super().\_\_init\_\_(参数) 显式调用父类的构造方法，否则父类的实例属性无法被初始化，子类实例将缺少父类的相关属性。

#### <2>示例：单继承与重写
```python
# 父类：Animal
class Animal:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def eat(self, food):
        print(f"{self.name}正在吃{food}")
    
    def move(self):
        print(f"{self.name}正在缓慢移动")

# 子类：Dog 继承 Animal
class Dog(Animal):
    # 扩展子类特有的属性（品种）
    def __init__(self, name, age, breed):
        # 关键：必须显式调用父类__init__，才能继承name和age属性
        # 若省略此句，Dog实例将缺少name和age属性，调用父类方法时会报错
        super().__init__(name, age)
        self.breed = breed  # 子类新增属性：品种
    
    # 重写父类的move方法（覆盖父类逻辑）
    def move(self):
        print(f"{self.name}（{self.breed}）正在快速奔跑")
    
    # 扩展子类特有的方法（叫）
    def bark(self):
        print(f"{self.name}汪汪叫！")

# 实例化子类Dog
dog = Dog("旺财", 3, "金毛")

# 访问父类的属性（因显式调用父类__init__，才可正常访问）
print(dog.name)  # 输出：旺财
print(dog.age)   # 输出：3

# 访问子类的属性
print(dog.breed)  # 输出：金毛

# 调用父类的方法（未被重写）
dog.eat("骨头")  # 输出：旺财正在吃骨头

# 调用重写后的方法（执行子类逻辑）
dog.move()  # 输出：旺财（金毛）正在快速奔跑

# 调用子类特有的方法
dog.bark()  # 输出：旺财汪汪叫！



# 反面示例：子类未调用父类__init__
class Cat(Animal):
    def __init__(self, color):
        self.color = color  # 仅定义子类属性，未调用父类__init__

cat = Cat("橘色")
# print(cat.name)  # 报错：AttributeError: 'Cat' object has no attribute 'name'
# cat.eat("鱼")    # 报错：AttributeError: 'Cat' object has no attribute 'name'
```

#### <3>示例：多继承（了解）
- 子类继承多个父类，语法为 **class 子类名(父类1, 父类2, ...)**
- 注意「方法解析顺序（MRO）」：
	- 当多个父类有同名方法时，子类按继承列表的顺序优先调用前面父类的方法。
	- 若要调用后面父类的方法，可通过类名直接调用并传递实例
```python
# 父类1
# 父类1：拥有自身属性和方法
class A:
    # 父类1的属性
    feature_a = "来自A类的特征"
    
    def show(self):
        print("A类的show方法")
    
    # 父类1的独有方法
    def func_a(self):
        print(f"调用A类独有方法，{self.feature_a}")

# 父类2：拥有自身属性和方法
class B:
    # 父类2的属性
    feature_b = "来自B类的特征"
    
    def show(self):
        print("B类的show方法")
    
    # 父类2的独有方法
    def func_b(self):
        print(f"调用B类独有方法，{self.feature_b}")

# 子类C继承A和B（顺序：A在前，B在后）
class C(A, B):
    # 子类可新增自身属性
    own_feature = "C类自身的特征"
    
    # 子类可新增自身方法
    def func_own(self):
        print(f"调用C类自身方法，{self.own_feature}")

# 实例化C
c = C()

# 1. 访问继承自A类的属性和独有方法
print(c.feature_a)  # 输出：来自A类的特征
c.func_a()          # 输出：调用A类独有方法，来自A类的特征

# 2. 访问继承自B类的属性和独有方法
print(c.feature_b)  # 输出：来自B类的特征
c.func_b()          # 输出：调用B类独有方法，来自B类的特征

# 3. 访问子类自身的属性和方法
print(c.own_feature)# 输出：C类自身的特征
c.func_own()        # 输出：调用C类自身方法，C类自身的特征

# 4. 调用同名方法show()，遵循MRO优先调用A类方法（A在继承列表前）
c.show()            # 输出：A类的show方法

# 5. 查看MRO（方法解析顺序）：C -> A -> B -> object
print(C.mro())  # 输出：[<class '__main__.C'>, <class '__main__.A'>, <class '__main__.B'>, <class 'object'>]

# 6. 若需调用B类的同名show方法，可通过类名直接调用并传递实例
B.show(c)  # 输出：B类的show方法
```

#### 补充1：父类各属性的继承情况
##### 1）类属性：完全继承
- 子类会直接继承父类的所有类属性（公开、单下划线保护的类属性）
- 通过 **「子类名.类属性名」** 或 **「子类实例.类属性名」** 直接 **访问**
	- 如果通过「子类名.类属性名」或「子类实例.类属性名」 **修改** ——>其实是**子类重写该属性**
- **修改父类的类属性会影响所有子类实例（子类未重写该类属性时）**。
```python
# 父类
class Person:
    # 类属性：公开属性+单下划线保护属性，均可被子类继承
    species = "人类"
    _feature = "语言能力"  # 保护类属性，子类仍可继承/访问

# 子类（继承父类）
class Student(Person):
    pass  # 无自定义属性，完全继承父类所有类属性

# 1. 子类/实例访问继承的类属性（含保护属性）
print(Student.species)       # 子类名访问公开类属性 → 人类
stu1 = Student()
print(stu1._feature)         # 实例访问继承的保护类属性 → 语言能力

# 2. 修改父类类属性，影响子类实例
Person.species = "智人"
print(stu1.species)          # 子类实例同步变化 → 智人

# 3. 子类创建了自己的同名类属性，遮蔽了父类的类属性
Student.species = "学生"
print(stu1.species)          # 子类实例用重写值 → 学生
print(Person.species)        # 父类属性不变 → 智人

# 4. 子类实例直接修改保护类属性（_feature），实际创建同名实例属性
stu1._feature = "逻辑思维能力"
print(stu1._feature)         # 实例属性覆盖 → 逻辑思维能力
print(Student._feature)      # 子类类属性仍为 → 语言能力
print(Person._feature)       # 父类类属性仍为 → 语言能力
```

##### 2）实例属性：需显式调用父类__init__才继承
- 父类的实例属性定义在__init__中
- **若子类自定义了\_\_init\_\_**：
	- 必须通过super().\_\_init\_\_(参数)**显式调用父类构造方法**，才能继承父类的实例属性
- **若子类未自定义\_\_init\_\_**：
	- 会自动继承父类的__init__，实例化时直接传入父类所需参数即可继承。
```python
# 父类：__init__中定义实例属性name、age
class Person:
    def __init__(self, name, age):
        self.name = name  # 父类实例属性1
        self.age = age    # 父类实例属性2

# 子类1：未自定义__init__ → 自动继承父类__init__
class Student1(Person):
    pass  

# 实例化子类1：直接传入父类所需参数，继承所有实例属性
stu1 = Student1("张三", 18)
print(f"Student1：{stu1.name}，{stu1.age}")  # 输出：张三，18

# 子类2：自定义__init__但未调用父类构造方法 → 不继承父类实例属性
class Student2(Person):
    def __init__(self, school):
        self.school = school  # 仅子类自身实例属性

# 实例化子类2：仅传子类参数，无法访问父类属性
stu2 = Student2("北京大学")
print(f"Student2：{stu2.school}")  # 输出：北京大学
# print(stu2.name)  # 报错：'Student2' object has no attribute 'name'

# 子类3：自定义__init__且显式调用父类构造方法 → 继承父类实例属性
class Student3(Person):
    def __init__(self, name, age, school):
        super().__init__(name, age)  # 显式调用父类__init__
        self.school = school        # 子类扩展实例属性

# 实例化子类3：传入父类+子类参数，可访问所有属性
stu3 = Student3("李四", 20, "清华大学")
print(f"Student3：{stu3.name}，{stu3.age}，{stu3.school}")  # 输出：李四，20，清华大学
```

##### 3）私有属性（双下划线__attr）：不直接继承，可通过父类公开方法间接访问
- 父类的双下划线私有属性会被 Python 通过**「名称修饰」机制**隐藏（修饰为\_父类名\_\_私有属性名）
- 子类无法直接通过原属性名访问，也不视为 “继承” 了该属性（**语法层面隔离**）。
- 但子类可通过父类提供的公开 getter/setter 方法间接操作父类的私有属性（**本质是父类内部允许访问，子类调用父类方法实现**）
```python
class Person:
    def __init__(self, name, age):
        self.name = name        # 公开属性
        self.__age = age        # 私有属性（名称修饰）
    
    # 公开方法：操作私有属性（函数体缩进4空格，核心规则）
    def get_age(self):
        return self.__age
    def set_age(self, new_age):
        self.__age = new_age

class Student(Person):
    def __init__(self, name, age, school):
        super().__init__(name, age)
        self.school = school

# 1. 子类无法直接访问父类私有属性
stu = Student("张三", 18, "北大")
# print(stu.__age)  # 报错：无__age属性

# 2. 子类通过父类公开方法间接操作
print(stu.get_age())  # 输出：18
stu.set_age(20)
print(stu.get_age())  # 输出：20

# 3. 验证名称修饰（仅理解，禁止使用）
print(stu._Person__age)  # 输出：20
```

#### 补充2：父类各类方法的继承情况
##### 1）实例方法（公开、单下划线保护）：完全继承
- 子类会继承父类的所有公开实例方法（无下划线）、单下划线保护实例方法，可通过子类实例直接调用
- 若子类定义了与父类同名的方法，会重写父类方法（调用时优先执行子类逻辑）。
```python
class Person:
    # 公开实例方法
    def say_hello(self):
        print("父类：你好！")
    
    # 单下划线保护实例方法
    def _show_info(self):
        print("父类：基础信息")

# 子类继承父类方法，并重写公开方法
class Student(Person):
    # 重写父类公开方法
    def say_hello(self):
        print("子类：同学你好！")

# 1. 子类实例调用继承的保护方法
stu = Student()
stu._show_info()  # 输出：父类：基础信息（继承保护方法）

# 2. 子类实例调用重写后的公开方法
stu.say_hello()   # 输出：子类：同学你好！（优先执行子类逻辑）
```

##### 2）类方法、静态方法：完全继承
- 子类会直接继承父类的公开 / 保护类方法（**@classmethod修饰**）、静态方法（**@staticmethod修饰**）
- 通过「**子类名.方法名**」或「**子类实例.方法名**」调用
- 子类可重写父类的类方法、静态方法。
```python
class Person:
    # 类方法
    @classmethod
    def class_func(cls):
        print("父类类方法")
    
    # 静态方法
    @staticmethod
    def static_func():
        print("父类静态方法")

class Student(Person):
    # 重写父类类方法
    @classmethod
    def class_func(cls):
        print("子类类方法")

# 1. 子类名调用继承的静态方法
Student.static_func()  # 输出：父类静态方法

# 2. 子类实例调用继承的静态方法
stu = Student()
stu.static_func()      # 输出：父类静态方法

# 3. 子类名调用重写后的类方法
Student.class_func()   # 输出：子类类方法

# 4. 子类实例调用重写后的类方法
stu.class_func()       # 输出：子类类方法
```

##### 3）私有方法（双下划线__method）：不直接继承，可通过父类公开方法间接调用
- 与私有属性类似，父类的双下划线私有方法会被名称修饰（修饰为 **\_父类名\_\_私有方法名**）
- 子类无法直接调用，只能**通过父类的公开方法间接触发**。
```python
class Person:
    def __init__(self):
        pass

    # 双下划线私有方法
    def __private_func(self):
        print("父类私有方法")

    # 公开方法：间接触发私有方法
    def call_private(self):
        self.__private_func()

class Student(Person):
    def try_call_private(self):
        # 直接调用父类私有方法会报错
        # self.__private_func()
        # 调用父类公开方法间接触发私有方法
        self.call_private()

# 1. 子类实例无法直接调用私有方法
stu = Student()
# stu.__private_func()  # 报错：无该属性

# 2. 子类通过父类公开方法间接触发私有方法
stu.try_call_private()  # 输出：父类私有方法

# 3. 验证名称修饰（仅理解，禁止使用）
stu._Person__private_func()  # 输出：父类私有方法
```

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111114413251.png)

### （3）多态
- **核心思想**：
	- 同一方法名在不同类中有不同的实现逻辑，当调用该方法时，根据对象的实际类型执行对应的逻辑（即“一个接口，多种实现”）。
	- 多态依赖于继承和重写，核心作用是提高代码的灵活性和可扩展性。
- **实现条件**：
	- 存在继承关系（子类继承父类）
	- 子类重写父类的方法
	- 使用**父类类型的变量引用子类对象**，**调用重写方法**时自动匹配子类逻辑。
```python
# 父类：Animal
class Animal:
    def __init__(self, name):
        self.name = name
    
    # 父类方法（被子类重写）
    def make_sound(self):
        print(f"{self.name}发出声音")

# 子类1：Dog（重写make_sound）
class Dog(Animal):
    def make_sound(self):
        print(f"{self.name}汪汪叫！")

# 子类2：Cat（重写make_sound）
class Cat(Animal):
    def make_sound(self):
        print(f"{self.name}喵喵叫！")

# 子类3：Duck（重写make_sound）
class Duck(Animal):
    def make_sound(self):
        print(f"{self.name}嘎嘎叫！")

# 多态核心：统一接口（调用make_sound），不同对象执行不同逻辑
def animal_sound(animal):
    # 接收Animal类型的参数（实际可传入任意子类对象）
    animal.make_sound()

# 实例化不同子类
dog = Dog("旺财")
cat = Cat("咪宝")
duck = Duck("小黄")

# 调用统一接口，自动匹配子类逻辑
animal_sound(dog)  # 输出：旺财汪汪叫！
animal_sound(cat)  # 输出：咪宝喵喵叫！
animal_sound(duck) # 输出：小黄嘎嘎叫！
```

## 5.类的高级用法
### （1）魔术方法（特殊方法）
Python类中**以 \_\_ 开头和结尾**的方法称为“魔术方法”（如__init__），它们在**特定场景下自动调用，无需手动调用**

#### <1>构造与析构相关
- **\_\_init\_\_(self, ...)**：构造方法，实例化对象时自动调用（初始化属性）。
- **\_\_del\_\_(self)**：：析构方法，对象被垃圾回收时自动调用（释放资源，如关闭文件）。
```python
class FileHandler:
    def __init__(self, filename):
        self.filename = filename
        self.file = open(filename, "r", encoding="utf-8")
        print(f"打开文件：{self.filename}")
    
    def read_content(self):
        return self.file.read()
    
    def __del__(self):
        self.file.close()
        print(f"关闭文件：{self.filename}")

# 实例化
fh = FileHandler("test.txt")
print(fh.read_content())
# 输出：（文件内容）
# 程序结束时自动输出：关闭文件：test.txt
```

#### <2>运算符重载相关
通过重写魔术方法，让自定义类的对象支持Python内置运算符（如 +、-、==、< 等）
```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    # 重载 + 运算符（实现两个Point对象相加）
    def __add__(self, other):
        return Point(self.x + other.x, self.y + other.y)
    
    # 重载 == 运算符（判断两个Point对象是否相等）
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
    
    # 重载字符串输出（print对象时显示自定义格式）
    def __str__(self):
        return f"Point({self.x}, {self.y})"

# 实例化
p1 = Point(1, 2)
p2 = Point(3, 4)

# 调用__add__（+运算符）
p3 = p1 + p2
print(p3)  # 输出：Point(4, 6)（调用__str__）

# 调用__eq__（==运算符）
print(p1 == p2)  # 输出：False
print(p1 == Point(1, 2))  # 输出：True
```
**想进一步了解，请参考:**[Python数据模型](/2026/01/09/Python/《流畅的python》/第一章：Python数据模型/)

### （2）类的组合 <a id="class-composition"></a>
- 组合是另一种代码复用方式，指**一个类的属性是另一个类的对象**（即“一个类包含另一个类”）
- 适用于“整体-部分”关系（如“汽车”包含“发动机”、“轮胎”）。组合比继承更灵活，避免了继承的耦合性。
```python
# 部分类1：发动机
class Engine:
    def start(self):
        print("发动机启动")
    
    def stop(self):
        print("发动机关闭")

# 部分类2：轮胎
class Tire:
    def __init__(self, brand):
        self.brand = brand
    
    def rotate(self):
        print(f"{self.brand}轮胎旋转")

# 整体类：汽车（组合Engine和Tire）
class Car:
    def __init__(self, tire_brand):
        # 组合：Car类包含Engine对象和Tire对象
        self.engine = Engine()
        self.tire1 = Tire(tire_brand)
        self.tire2 = Tire(tire_brand)
    
    def run(self):
        self.engine.start()
        self.tire1.rotate()
        self.tire2.rotate()
        print("汽车行驶中...")

# 实例化汽车
car = Car("米其林")
car.run()
# 输出：
# 发动机启动
# 米其林轮胎旋转
# 米其林轮胎旋转
# 汽车行驶中...
```

### （3）抽象类（ABC模块）
- 抽象类是一种“**不能实例化**”的类，用于定义子类必须实现的方法（规范子类接口）。
- Python通过 abc 模块实现抽象类
- 核心作用：是强制子类遵循统一的接口规范。
```python
from abc import ABC, abstractmethod

# ABC 是 Abstract Base Class 的缩写，且 Python 3.4+ 推荐直接继承 ABC，而非使用 metaclass=ABCMeta

# 抽象类（继承ABC）
class Shape(ABC):
    # 抽象方法（子类必须重写）
    @abstractmethod
    def area(self):
        pass
    
    # 抽象方法（子类必须重写）
    @abstractmethod
    def perimeter(self):
        pass

# 子类1：圆形（实现抽象方法）
class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    
    def area(self):
        return 3.14 * self.radius ** 2
    
    def perimeter(self):
        return 2 * 3.14 * self.radius

# 子类2：正方形（实现抽象方法）
class Square(Shape):
    def __init__(self, side):
        self.side = side
    
    def area(self):
        return self.side ** 2
    
    def perimeter(self):
        return 4 * self.side

# 抽象类不能实例化（报错）
# shape = Shape()  # TypeError: Can't instantiate abstract class Shape with abstract method area, perimeter

# 子类可实例化（已实现所有抽象方法）
circle = Circle(5)
print(f"圆形面积：{circle.area()}, 周长：{circle.perimeter()}")  # 输出：圆形面积：78.5, 周长：31.4

square = Square(4)
print(f"正方形面积：{square.area()}, 周长：{square.perimeter()}")  # 输出：正方形面积：16, 周长：16
```

## 6.导入类
在 Python 中，“导入类”是实现代码模块化复用的核心手段，通过将不同功能的类拆分到不同“**模块**”（**.py 文件**），可大幅提升代码的可维护性、可读性和复用性。

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111154058716.png)

### （1）核心概念
- **模块（module）**：
	- 本质是一个 **.py 后缀**的 Python 文件，可包含类、函数、变量等代码片段，是代码组织的基本单位。
		- 例如：animal.py 就是一个模块
- **包（package）**：
	- **包含多个模块的目录**，需包含 **\_\_init\_\_.py** 文件（Python 3.3+ 可省略，但推荐保留），用于管理多模块层级结构。
		- 例如 my_package/ 目录下包含 animal.py、person.py，则 my_package 是包
- **导入核心目的**：
	- 在当前模块中复用其他模块已定义的类，避免重复编码；
	- 实现代码功能拆分，让不同模块负责不同业务逻辑。

### （2）导入单个类
适用于只需使用目标模块中某一个特定类的场景，语法简洁，导入后可直接使用类名调用。
#### <1>语法格式
```python
from 模块名 import 类名
```

#### <2>实战示例
- 步骤1：定义模块（被导入方）—— 新建animal.py文件，定义单个类
```python
# animal.py（模块）
class Animal:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def eat(self, food):
        print(f"{self.name}（{self.age}岁）正在吃{food}")
```
- 步骤2：导入并使用（导入方）—— 新建 main.py，从 animal.py 中导入 Animal 类
```python
# main.py（当前模块）
from animal import Animal  # 从 animal 模块导入 Animal 类

# 直接使用导入的类实例化
dog = Animal("旺财", 3)
dog.eat("骨头")  # 输出：旺财（3岁）正在吃骨头
```

### （3）在一个模块中存储多个类
适用于多个类功能关联紧密（如同一业务领域的不同实体）的场景，将相关类集中存储在一个模块中，便于管理和维护。

#### <1>核心原则
模块内的类需功能相关，避免将无关联的类随意放在同一模块（如将“动物类”和“数据库操作类”放在同一模块会导致混乱）。

#### <2>实战示例
- 步骤1：模块中定义多个关联类—— 修改 animal.py，新增 Dog、Cat 类（均继承 Animal）
```python
# animal.py（存储多个关联类）
class Animal:  # 父类
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def eat(self, food):
        print(f"{self.name}（{self.age}岁）正在吃{food}")

class Dog(Animal):  # 子类1：狗
    def bark(self):
        print(f"{self.name}汪汪叫！")

class Cat(Animal):  # 子类2：猫
    def meow(self):
        print(f"{self.name}喵喵叫！")
```
- 步骤2：导入使用模块中的多个类 —— 在 main.py 中可按需导入其中一个或多个类。（见下文）


### （4）从一个模块中导入多个类
适用于需要使用目标模块中多个类的场景，通过逗号分隔类名，一次性导入多个所需类。
#### <1>语法格式
```python
from 模块名 import 类名1, 类名2, 类名3
```

#### <2>实战演示
```python
# main.py
from animal import Dog, Cat  # 从 animal 模块导入 Dog 和 Cat 两个类

# 使用导入的多个类
dog = Dog("旺财", 3)
dog.eat("骨头")  # 输出：旺财（3岁）正在吃骨头
dog.bark()       # 输出：旺财汪汪叫！

cat = Cat("咪宝", 2)
cat.eat("鱼")    # 输出：咪宝（2岁）正在吃鱼
cat.meow()       # 输出：咪宝喵喵叫！
```

### （5）导入整个模块
- 适用于需要使用目标模块中多个类，且希望区分“类所属模块”的场景（避免类名冲突）。
- 导入后需通过“**模块名.类名**”的方式访问类。
- **核心优势**：明确类的来源，避免不同模块中同名类的冲突（如同时导入两个模块的 Person 类，可通过 module1.Person 和 module2.Person 区分）
#### <1>语法格式
```python
import 模块名
```

#### <2>实战演示
```python
# main.py
import animal  # 导入整个 animal 模块

# 通过“模块名.类名”访问模块中的类
animal_obj = animal.Animal("通用动物", 1)
animal_obj.eat("食物")  # 输出：通用动物（1岁）正在吃食物

dog = animal.Dog("小黑", 2)
dog.bark()  # 输出：小黑汪汪叫！
```

### （6）导入模块中的所有类（不推荐）
- 适用于需要使用目标模块中绝大多数或全部类的场景，但**因易导致“命名污染”（导入冗余类）和“类名冲突”**
- 仅推荐在小型脚本中使用，**强烈不推荐在正式项目**中使用
	- 不清楚导入了哪些名称
	- 可能覆盖当前模块已有的同名变量/函数/类
#### <1>语法格式
```python
from 模块名 import *
```

#### <2>实战示例
```python
# main.py
from animal import *  # 导入 animal 模块中的所有类

# 直接使用模块中的所有类（无需加模块名前缀）
animal_obj = Animal("通用动物", 1)
dog = Dog("旺财", 3)
cat = Cat("咪宝", 2)

animal_obj.eat("食物")  # 输出：通用动物（1岁）正在吃食物
dog.bark()              # 输出：旺财汪汪叫！
cat.meow()              # 输出：咪宝喵喵叫！
```

### （7）自定义导入（别名、包层级导入、路径配置）
针对特殊需求（如**类名/模块名过长**、**同名冲突**、**跨目录导入**）的自定义导入方式，是实际开发中高频使用的进阶技巧。

#### <1>给类/模块指定别名（解决命名冲突/简化名称）
当类名/模块名过长，或不同模块存在同名类时，通过 as 关键字指定别名，提升代码简洁性并避免冲突。
```python
# 语法
# 1.给类指定别名
from 模块名 import 类名 as 别名
# 2.给模块指定别名
import 模块名 as 模块别名
```
```python
# ----------------- 给类指定别名 ----------------
# 模块1：person1.py
class Person:
    def __init__(self, name):
    	self.name = name
        print(f"来自person1的{name}")

# 模块2：person2.py
class Person:
    def __init__(self, name):
        print(f"来自person2的{name}")

# 导入时指定别名
from person1 import Person as P1
from person2 import Person as P2

p1 = P1("张三")  # 输出：来自person1的张三
p2 = P2("李四")  # 输出：来自person2的李四



# ---------------- 给模块指定别名 ---------------
import very_long_module_name as vlmn  # 给长模块名指定短别名

# 通过模块别名访问类，不直接通过别名变量导入
obj = vlmn.VeryLongClassName()
obj.show()  # 简洁调用

```
**注！！**：不能通过模块别名变量导入类（Python无法识别）
```python
# 错误写法：试图通过模块别名变量导入类（Python无法识别）
# from vlmn import VeryLongClassName  # 报错：ModuleNotFoundError: No module named 'vlmn'
```


#### <2>包层级导入（跨目录导入）
当类所在的模块位于包（目录）中时，需通过“包名.模块名”的层级路径导入，适用于大型项目的多目录结构管理。

**目录示例**：
```text
my_project/          # 项目根目录
├── main.py          # 当前模块（导入方）
└── my_package/      # 包目录
    ├── __init__.py  # 包初始化文件（可为空）
    ├── animal.py    # 模块1（含 Animal、Dog、Cat 类）
    └── person.py    # 模块2（含 Person 类）
```

**导入语法**：
```python
# 方式1：从包的模块中导入单个类
from my_package.animal import Dog

# 方式2：导入包的模块，再通过模块访问类
import my_package.person as mp_person

# 方式3：从包中导入整个模块
from my_package import animal
```

**使用示例**：
```python
# my_package/animal.py
class Animal:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def eat(self, food):
        print(f"{self.name}（{self.age}岁）正在吃{food}")

class Dog(Animal):
    def bark(self):
        print(f"{self.name}汪汪叫！")

class Cat(Animal):
    def meow(self):
        print(f"{self.name}喵喵叫！")
```
```python
# my_package/person.py
class Person:
    def __init__(self, name, pet):
        self.name = name
        self.pet = pet
    
    def play_with_pet(self):
        print(f"{self.name} 和 {self.pet.name} 一起玩耍")
```
```python
# main.py
from my_package.animal import Dog
import my_package.person as mp_person
from my_package import animal

# 使用方式1导入的 Dog 类
dog = Dog("旺财", 3)
dog.bark()  # 输出：旺财汪汪叫！

# 使用方式2导入的模块访问 Person 类
person = mp_person.Person("张三", dog)
person.play_with_pet()  # 输出：张三 和 旺财 一起玩耍

# 使用方式3导入的模块访问类
cat = animal.Animal("咪咪", 2)
cat.eat("鱼")  # 输出：咪咪（2岁）正在吃鱼
```

#### <3>自定义模块路径（导入非项目根目录的模块）
需要导入的模块不在 Python 默认搜索路径（sys.path）中时，需手动将模块所在目录添加到搜索路径，再进行导入。

**目录示例**：
```
/home/user/
├── my_project/		 # 项目根目录
│   └── main.py          # 你的主程序
│
└── shared_libs/         # 项目外部的公共库（不在 my_project 下）
    └── utils.py         # 想导入的模块
```

**问题**：
- 直接在 main.py 中 import utils 会报错,因为 shared_libs/ 不在 Python 的搜索路径中。
```python
# main.py
import utils  # ❌ ModuleNotFoundError: No module named 'utils'
```

**解决方案**：
```python
# main.py
import sys

# 手动添加外部模块所在目录
sys.path.append("/home/user/shared_libs")

# 现在可以导入了
import utils
utils.helper()  # ✅ 正常调用
```

### （8）注意事项
#### <1>概述
- **避免循环导入**：
	- 禁止出现“模块 A 导入模块 B 的类，模块 B 又导入模块 A 的类”的循环依赖，会直接导致导入失败（ImportError）。
	- **解决方案**：
		- 将共同依赖的类提取到第三方模块，或延迟导入（在函数内部导入）。
- **优先明确导入**：
	- 推荐使用“from 模块名 import 类名”（导入单个/多个类）或“import 模块名”（导入整个模块）
	- 避免使用“from 模块名 import \*”，减少命名污染和冲突风险。
- **模块路径规范**：
	- 同目录导入：直接使用模块名（如 from animal import Animal）。
- **子目录导入**：
	- 通过包名+模块名（如 from my_package.animal import Animal）。
- **上级目录导入**：
	- 需手动添加路径到 sys.path（如 sys.path.append("../")）。
- **类名冲突处理**：
	- 当导入的类与当前模块的类同名时，“后导入/定义的类”会覆盖“先导入/定义的类“。
	- **解决方案**：
		- 使用**别名（as）** 或通过 **“模块名.类名”**访问。

#### <2> 循环导入问题与解决方案
- **错误示例**
```python
# ---------- 错误示例：循环导入导致模块加载失败 ----------

# 模块 A：a.py
from b import B  # 模块加载时立即导入 B

class A:
    def __init__(self):
        self.b = B()
    
    def show(self):
        print(f"A 持有 B: {self.b}")

# 模块 B：b.py
from a import A  # 模块加载时立即导入 A（形成循环）

class B:
    def __init__(self):
        self.a = A()
    
    def show(self):
        print(f"B 持有 A: {self.a}")

# main.py
from a import A  # ❌ 报错

# ImportError: cannot import name 'B' from partially initialized module 'b'
# (most likely due to a circular import)
```

- **正确解决方案1**：合并到同一模块
```python
# 模块：entities.py（将相互依赖的类放在同一模块）
class A:
    def __init__(self):
        self.b = None  # 先不初始化
    
    def set_b(self, b):
        self.b = b

class B:
    def __init__(self):
        self.a = None
    
    def set_a(self, a):
        self.a = a

# 使用方：main.py
from entities import A, B

a = A()
b = B()
a.set_b(b)  # 手动关联
b.set_a(a)

print(a.b)  # <entities.B object ...>
print(b.a)  # <entities.A object ...>
```

- **正确解决方案1（变体）**：通过工厂/协调模块解耦
```python
# 模块 A：a.py
class A:
    def __init__(self):
        self.b = None  # 不在构造函数中创建 B
    
    def link(self, b):
        self.b = b

# 模块 B：b.py
class B:
    def __init__(self):
        self.a = None
    
    def link(self, a):
        self.a = a

# 协调模块：factory.py（负责创建和关联）
from a import A
from b import B

def create_linked_pair():
    """创建互相关联的 A 和 B 实例"""
    a = A()
    b = B()
    a.link(b)
    b.link(a)
    return a, b

# 使用方：main.py
from factory import create_linked_pair

a, b = create_linked_pair()
print(a.b)  # <b.B object ...>
print(b.a)  # <a.A object ...>
```


- **正确解决方案2**：延迟导入（函数内部导入）
```python
# ---------- 正确解决方案2：延迟导入（函数内部导入） ----------

# 模块 A：a.py
class A:
    def __init__(self):
        self.b = None
    
    def create_b(self):
        from b import B
        self.b = B()
        return self.b
    
    def show(self):  # 补充 show 方法
        print(f"A 持有 B: {self.b}")

# 模块 B：b.py
class B:
    def __init__(self):
        self.a = None
    
    def create_a(self):
        from a import A
        self.a = A()
        return self.a
    
    def show(self):  # 补充 show 方法
        print(f"B 持有 A: {self.a}")

# 使用方：main.py
from a import A  # 补充导入
a = A()
b = a.create_b()
b.a = a  # 手动反向关联

a.show()  # 输出：A 持有 B: <b.B object at 0x...>
b.show()  # 输出：B 持有 A: <a.A object at 0x...>
```

- **正确解决方案2（变体）**：用于类型注解的延迟导入
```python
# 模块 A：a.py
from __future__ import annotations  # 启用延迟注解求值
from typing import TYPE_CHECKING

if TYPE_CHECKING:  # 仅在类型检查时导入，运行时不执行
    from b import B

class A:
    def __init__(self):
        self.b: B | None = None  # 类型注解可用，但运行时不导入
    
    def set_b(self, b: B):
        self.b = b

# 模块 B：b.py
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from a import A

class B:
    def __init__(self):
        self.a: A | None = None
    
    def set_a(self, a: A):
        self.a = a

# 使用方：main.py
from a import A
from b import B

a = A()
b = B()
a.set_b(b)
b.set_a(a)

print(f"a.b = {a.b}")  # 输出正常
print(f"b.a = {b.a}")  # 输出正常

```

### 补充：\_\_init\_\_.py 文件 的使用
在包目录中，\_\_init\_\_.py 是标识目录为 Python 包的核心文件（Python 3.3+ 支持隐式命名空间包，可省略，但工程化开发中推荐保留），核心作用是简化导入和控制包的暴露内容，具体用法分为3类。

#### <1>基础用法：空文件（仅标识包)
只需在包目录下创建一个空的 \_\_init\_\_.py 文件，即可让 Python 将该目录识别为包，支持基础的包层级导入。
```
# 项目结构（空 __init__.py 示例）
my_package/          # 包目录
├── __init__.py      # 空文件（核心：标识为Python包）
├── animal.py        # 含 Animal、Dog 类
└── person.py        # 含 Person 类
```
```python
# 导入使用（空 __init__.py 支持的基础导入）
from my_package.animal import Dog
dog = Dog("旺财", 3)
dog.bark()  # 正常运行
```

#### <2>进阶用法1：简化导入路径（常用）
通过在 \_\_init\_\_.py 中提前导入包内模块的类/函数，可让外部导入时跳过中间模块，直接从包名导入目标类，大幅简化导入语法。
```
# 项目结构（空 __init__.py 示例）
my_package/          # 包目录
├── __init__.py      # 空文件（核心：标识为Python包）
├── animal.py        # 含 Animal、Dog 类
└── person.py        # 含 Person 类
```
```python
# 第一步：编写 my_package/__init__.py
from .animal import Animal, Dog  # 从当前包的 animal 模块导入指定类
from .person import Person       # 从当前包的 person 模块导入指定类
```
```python
# 第二步：外部导入（简化前 vs 简化后）
# 简化前（需写全模块路径）
from my_package.animal import Dog
from my_package.person import Person

# 简化后（直接从包名导入，跳过模块名）
from my_package import Dog, Person
dog = Dog("旺财", 3)
person = Person("张三", dog)
person.play_with_pet()  # 正常运行
```

#### <3>进阶用法2：控制“from 包名 import \*”的导入内容
通过 \_\_all\_\_ 变量（列表类型），可明确指定当外部使用 from 包名 import \* 时，能导入的类/模块列表，避免导入冗余内容，减少命名污染。
```
# 项目结构（空 __init__.py 示例）
my_package/          # 包目录
├── __init__.py      # 空文件（核心：标识为Python包）
├── animal.py        # 含 Animal、Dog 类
└── person.py        # 含 Person 类
```
```python
# 第一步：编写 my_package/__init__.py
__all__ = ["Animal", "Person"]  # 仅允许外部通配导入这两个类
from .animal import Animal, Dog  # 包内导入了 Dog，但未加入 __all__
from .person import Person
```
```python
# 第二步：外部通配导入测试
from my_package import *  # 仅能导入 __all__ 中的 Animal 和 Person

# 可正常使用
animal = Animal("通用动物", 1)
person = Person("张三", animal)

# 无法使用（Dog 未在 __all__ 中，通配导入未包含）
# dog = Dog("旺财", 3)  # 报错：NameError: name 'Dog' is not defined

# 若需使用 Dog，需明确导入
from my_package.animal import Dog
dog = Dog("旺财", 3)  # 正常运行
```

### 补充：\_\_name\_\_ == "\_\_main\_\_"
Python 中 \_\_name\_\_ == "\_\_main\_\_" 的核心作用，是通过 \_\_name\_\_ 变量的动态取值，区分模块的两种核心使用状态——**「直接使用」**和**「被导入」**，进而**实现代码的“按需执行”**。

#### 1）\_\_name\_\_ 是什么？
\_\_name\_\_ 是 Python 为每个 .py 文件（模块）内置的 **「模块级特殊变量」**，无需手动定义，其值由模块的“使用方式”决定：
- 当模块 **直接使用** 时：
	- **\_\_name\_\_ = "\_\_main\_\_"**（\_\_main\_\_ 是 Python 规定的“主模块”标识）
- 当模块 **被导入** 时：
	- \_\_name\_\_ = 模块名（即**模块的文件名**，不含 .py 后缀）

\_\_name\_\_ == "\_\_main\_\_" 本质是一个“状态判断语句”，用于检测当前模块是否处于「直接使用」状态。

#### 2）模块被直接使用（作为主程序入口）
当我们通过命令行执行 **python 模块名.py** ，或在 IDE 中**直接运行某个 .py 文件**时，该模块就是“直接使用”的主模块，此时：
- 模块的 \_\_name\_\_ 自动赋值为 "\_\_main\_\_"；
- \_\_name\_\_ == "\_\_main\_\_" 条件成立，会执行该判断语句下的代码块。

**注！！**：
- **这里的“主程序入口”，是指"判断当前模块是否是用户直接运行的主模块"，而不是“执行入口”**

**示例代码**：
```python
# calculator.py

def add(a, b):
    """加法函数"""
    return a + b

def multiply(a, b):
    """乘法函数"""
    return a * b

# 查看当前模块的 __name__ 值
print(f"当前模块的 __name__ = {__name__}")

# 判断是否为直接运行
if __name__ == "__main__":
    # 只有直接运行 calculator.py 时，这里的代码才会执行
    print("=" * 40)
    print("模块被直接运行，执行测试代码：")
    print(f"add(10, 20) = {add(10, 20)}")
    print(f"multiply(5, 6) = {multiply(5, 6)}")
    print("=" * 40)
```

**直接运行**：
```text
python calculator.py
```

**输出**：
```text
当前模块的 __name__ = __main__
========================================
模块被直接运行，执行测试代码：
add(10, 20) = 30
multiply(5, 6) = 30
========================================

```

#### 3）模块被导入使用
当我们在 A 模块中通过import B 导入 B 模块时，B 模块是“被导入”的工具库模块，此时：
- B 模块的 \_\_name\_\_ 自动赋值为其模块名（如 "direct_use_demo"）；
- B 模块中 \_\_name\_\_ == "\_\_main\_\_" 条件不成立，不会执行该判断下的代码块。

**文件结构**
```text
project/
├── utils.py      # 被导入的工具模块（B模块）
└── main.py       # 主程序（A模块）
```

**utils.py（被导入的模块）**
```python
# utils.py

def greet(name):
    """问候函数"""
    return f"Hello, {name}!"

def calculate_area(radius):
    """计算圆的面积"""
    import math
    return math.pi * radius ** 2

# 查看当前模块的 __name__ 值
print(f"[utils.py] 当前 __name__ = {__name__}")

# 这段代码只有直接运行 utils.py 时才执行
if __name__ == "__main__":
    print("=" * 40)
    print("utils.py 被直接运行，执行测试代码：")
    print(greet("World"))
    print(f"半径为 5 的圆面积：{calculate_area(5):.2f}")
    print("=" * 40)
```

**运行 main.py（utils 被导入）**
```text
python main.py
```

**输出**：
```python
[main.py] 程序开始
[main.py] 当前 __name__ = __main__
----------------------------------------
[utils.py] 当前 __name__ = utils
----------------------------------------
[main.py] 导入完成，开始调用函数
Hello, Python!
半径为 10 的圆面积：314.16
```



# 十、文件
- Python 文件操作的核心是通过内置函数（如 **open()**）和标准库（如 **json**、pickle）实现数据的读取、写入与结构化存储。

## 1.open()函数
- Python 操作文件的**基础入口**是 open() 函数，该函数返回文件对象，通过文件对象的方法实现读写
- **核心语法**：
	- **open(file, mode, encoding, ...)**
		- **mode**（打开模式）：决定操作类型
		- **encoding**（编码）：仅针对文本文件
		- **返回值**：文件对象
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111171120316.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260112011135864.png)


## 2.with语句
### （1）基本语法与执行流程
```python
with open(file, mode, encoding="编码") as 文件对象:
    # 文件操作代码
    内容 = 文件对象.read()

# with 块结束后，文件自动关闭
```
```python
with open("test.txt", "r", encoding="utf-8") as f:
     │                                         	│
     │ ① 调用 open() 返回文件对象              	│
     │ ② 调用文件对象的 __enter__() 方法       	│
     │ ③ 将返回值赋给 f                        	│
     ▼                                         	│
    content = f.read()  ◄─── ④ 执行 with 块内代码
     │
     │ ⑤ with 块结束（正常/异常）
     ▼
    自动调用 f.__exit__() ──► 内部执行 f.close()

```

### （2）同时打开多个文件
#### <1>括号换行（Python 3.10+ 推荐！！）
```python
with (
    open("file1.txt", "r", encoding="utf-8") as f1,
    open("file2.txt", "r", encoding="utf-8") as f2,
    open("output.txt", "w", encoding="utf-8") as f_out,
):
    content1 = f1.read()
    content2 = f2.read()
    f_out.write(content1 + "\n" + content2)
```

#### <2>单行写法（逗号分隔）
```python
with open("input.txt", "r", encoding="utf-8") as f_in, open("output.txt", "w", encoding="utf-8") as f_out:
    content = f_in.read()
    f_out.write(content.upper())
# 两个文件都会自动关闭
```
#### <3>反斜杠换行（提高可读性）
```python
with open("input.txt", "r", encoding="utf-8") as f_in, \
     open("output.txt", "w", encoding="utf-8") as f_out:
    for line in f_in:
        f_out.write(line.upper())
```

#### <4>嵌套写法（兼容所有版本）
```python
with open("input.txt", "r", encoding="utf-8") as f_in:
    with open("output.txt", "w", encoding="utf-8") as f_out:
        f_out.write(f_in.read())
```


### （3）使用 vs 不使用 with
- **使用with**：
	- 能自动、安全地管理资源生命周期（无需手动关闭，即使代码抛出异常也会保证资源释放）
- **不使用with**：
	- 要么容易因忘记关闭导致资源泄漏，要么需手动写 try-finally 保证安全（代码繁琐），或仅简单手动关闭（异常场景下资源无法释放），既增加开发成本，又存在资源管理漏洞。
```python
# 方式1：忘记关闭（资源泄漏）
f = open("test.txt", "r", encoding="utf-8")
content = f.read()
# 忘记 f.close()，文件句柄未释放

# 方式2：手动关闭（异常不安全）
f = open("test.txt", "r", encoding="utf-8")
content = f.read()
f.close()  # 如果 read() 抛异常，close() 不会执行

# 方式3：try-finally（繁琐但安全）
f = open("test.txt", "r", encoding="utf-8")
try:
    content = f.read()
finally:
    f.close()  # 无论是否异常都会执行
```

### （4）with 语句不会创建新的作用域
with 不创建新作用域，变量属于外层作用域，只是文件被关闭了。
```python
with open("test.txt", "r", encoding="utf-8") as f:
    content = f.read()
    lines = f.readlines()  # 注意：此时已读完，lines 为空

# with 块外
print(content)      # ✅ 可用，content 仍存在
print(f)            # ✅ f 仍存在
print(f.closed)     # ✅ 输出 True（文件已关闭）
# f.read()          # ❌ 报错：I/O operation on closed file
```


## 3.读取文件
- 读取文件的核心逻辑：用 open() 以“读模式”打开文件，通过文件对象的读取方法获取数据，最后关闭文件
- **推荐用 with 上下文管理器自动关闭！**

### （1）read(size) ：读取指定长度/全部内容
- **读取指定长度/全部内容**
- **参数 size**（可选）： 
	- 指定读取的字节数
	- 不指定时读取文件全部内容
```python
# 语法示例1：读取全部文本内容（应用：内容筛选与保存）
with open("test.txt", "r", encoding="utf-8") as f:
    content = f.read()  # 读取全部内容

# 实际应用：筛选包含"Python"的句子，写入新文件
filtered_content = [sentence for sentence in content.split("\n") if "Python" in sentence]
with open("python_related.txt", "w", encoding="utf-8") as f:
    f.write("\n".join(filtered_content))

# 语法示例2：读取前 10 个字符（应用：获取文件头部信息）
with open("test.txt", "r", encoding="utf-8") as f:
    header = f.read(10)  # 仅读取前10个字符

# 实际应用：判断文件是否为指定类型（如判断日志文件头部标识）
if header.startswith("LOG-2024"):
    print("这是2024年的日志文件")
else:
    print("非目标日志文件")
```

### （2）readline()：按行读取（一次一行）
- **按行读取（一次一行）**
- 适用于大文件（避免一次性加载全部内容占用内存），每次调用读取一行，直到文件末尾返回空字符串
```python
# 语法示例1：逐行读取文本（仅演示）
with open("test.txt", "r", encoding="utf-8") as f:
    while True:
        line = f.readline()  # 一次读一行
        if not line:  # 读到末尾，返回空字符串，退出循环
            break
        print(line.strip())  # strip() 去除换行符和空格

# 方式2：直接迭代文件对象（更推荐）
with open("test.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())


# 语法示例2：逐行读取文本（应用：统计特定行数量）（仅演示）
count = 0
target_keyword = "错误"
with open("large_log.txt", "r", encoding="utf-8") as f, \
     open("error_log.txt", "w", encoding="utf-8") as err_f:
    line_num = 1  # 手动维护行号（enumerate的替代方案）
    # 用readline()循环读取，直到返回空字符串（文件末尾）
    line = f.readline()
    while line:  # 非空字符串为True，空字符串（EOF）为False
        if target_keyword in line:
            count += 1
            # 写入带行号的错误行，保持和原代码一致的格式
            err_f.write(f"行{line_num}：{line.strip()}\n")
        # 读取下一行，并更新行号
        line = f.readline()
        line_num += 1

print(f"日志中包含'错误'的行数：{count}")

#  方式2：直接迭代文件对象（更推荐）
with open("large_log.txt", "r", encoding="utf-8") as f, \
     open("error_log.txt", "w", encoding="utf-8") as err_f:  # 同时打开两个文件
    for line_num, line in enumerate(f, 1):
        if target_keyword in line:
            count += 1
            err_f.write(f"行{line_num}：{line.strip()}\n")

print(f"日志中包含'错误'的行数：{count}")
```

### （3）readlines()：读取所有行,返回列表
一次性读取文件所有行，每一行作为列表的一个元素，适用于中小文件。
```python
# 语法示例1：读取所有行并遍历
with open("test.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()  # 返回列表，元素为每一行内容
for line in lines:
    print(line.strip())


# 语法示例2：读取所有行并遍历（应用：数据清洗与格式转换）
with open("data.txt", "r", encoding="utf-8") as f:
    lines = f.readlines()  # 返回列表，元素为每一行内容

# 步骤1：清洗空行和空格
cleaned_lines = [line.strip() for line in lines if line.strip()]

# 步骤2：格式转换（将"姓名:年龄"格式转为"姓名,年龄"的CSV格式）
csv_lines = [line.replace(":", ",") for line in cleaned_lines]

# 步骤3：写入CSV文件
with open("data.csv", "w", encoding="utf-8") as f:
    f.write("姓名,年龄\n")
    f.writelines([line + "\n" for line in csv_lines])
```

### （4）二进制文件读取（如图片、视频）
二进制文件需用**“rb”模式**打开，读取的内容为字节串（bytes 类型），无需指定 encoding。
```python
# 语法示例1：文件复制（实际应用：备份图片/视频）
with open("image.jpg", "rb") as f:
    img_data = f.read()  # 读取二进制数据（bytes 类型）
# 写入新文件实现备份，保留原始格式
with open("image_backup.jpg", "wb") as f:
    f.write(img_data)
print("图片备份完成")

# 语法示例2：验证文件格式（实际应用：判断是否为JPG图片）
with open("unknown_file.bin", "rb") as f:
    # 检查文件头（前 2 字节）
    header = f.read(2)
    
    # 检查文件尾（最后 2 字节）
    f.seek(-2, 2)  # 从文件末尾向前移动 2 字节
    footer = f.read(2)

# JPG 文件：头部 0xFFD8，尾部 0xFFD9
if header == b'\xff\xd8' and footer == b'\xff\xd9':
    print("该文件是完整的 JPG 图片格式")
elif header == b'\xff\xd8':
    print("该文件可能是 JPG 图片（头部匹配，但尾部不完整）")
else:
    print("非 JPG 图片格式")

# 语法示例3：截取部分字节数据（实际应用：提取图片缩略信息）
with open("video.mp4", "rb") as f:
    # 读取前100字节数据（MP4文件头部包含格式信息）
    video_header = f.read(100)
# 将截取的头部信息写入新文件，用于格式分析
with open("video_header.bin", "wb") as f:
    f.write(video_header)
print("视频头部信息提取完成")
```

### 补充：读取结果content
#### <1>content的类型
**1.文本模式（r、r+）**：
- content 是 **str 类型（字符串）** ——> 操作方式就是操作“字符串”
- 包含文件中的原始字符（如换行符 \n、空格等）

**2.二进制模式（rb、rb+）**：
- content 是 **bytes 类型（字节串）**
- 格式为 b"xxx"，不涉及编码转换

#### <2>content的生命周期
content不是缩进在with块中定义吗，为什么with语句结束后仍能使用？
- **with 语句不会创建新的作用域！！**
	- 所以 with 块内定义的变量，在块外仍属于同一作用域，可以继续访问
```
with open(...) as f:
    content = f.read()
    │
    └── content 定义在 with 块内
        但 with 不创建新作用域
        ↓
        content 属于外层作用域（通常是函数/模块级）
        ↓
        with 结束后仍可访问 content
```
```python
# 示例1：with 块内的变量在块外可用
with open("test.txt", "r", encoding="utf-8") as f:
    content = f.read()
    line_count = 10

print(content)      # ✅ 正常
print(line_count)   # ✅ 正常，输出 10
print(f.closed)     # ✅ 输出 True（文件已关闭，但 f 变量仍存在）

# 示例2：for 循环同理
for i in range(5):
    x = i * 2

print(i)  # ✅ 输出 4（循环变量在循环外可用）
print(x)  # ✅ 输出 8
```

## 4.写入文件
- 写入文件的核心逻辑：用 open() 以“写模式”打开文件，通过文件对象的写入方法写入数据，
- **注意**：“写模式”会覆盖原有内容，“追加模式”则在文件末尾添加。

### （1）write(string/bytes)：写入字符串/字节串
- 文本模式下写入字符串，二进制模式下写入字节串
- 返回写入的字符/字节数。
```python
# 语法示例1：文本写入（覆盖原有内容，模式 "w"）
with open("test.txt", "w", encoding="utf-8") as f:
    count = f.write("Hello Python!\n")  # 写入字符串，\n 表示换行
    f.write("这是第二行内容")
print(f"写入的字符数：{count}")  # 输出：写入的字符数：14

# 语法示例2：文本追加（模式 "a"，在末尾添加，不覆盖原有内容）
with open("test.txt", "a", encoding="utf-8") as f:
    f.write("\n这是追加的内容")

# 语法示例3：二进制写入（模式 "wb"）
with open("data.bin", "wb") as f:
    f.write(b"Hello Binary!")  # 写入字节串，前缀 b 表示 bytes 类型

# 补充示例4：读取文本文件内容，写入另一个文本文件（文本文件复制/格式转换）
# 场景：读取源文件，过滤空行后写入目标文件
with open("source.txt", "r", encoding="utf-8") as read_f, open("target.txt", "w", encoding="utf-8") as write_f:
    for line in read_f:  # 逐行读取源文件，避免一次性加载大文件
        if line.strip():  # 过滤空行（strip() 去除空白字符后判断是否为空）
            write_f.write(line)  # 将非空行写入目标文件
print("文本文件内容过滤并写入完成")

# 补充示例5：读取二进制文件内容，写入另一个二进制文件（文件备份）
# 场景：备份图片、视频等二进制文件
with open("original.jpg", "rb") as read_f, open("backup.jpg", "wb") as write_f:
    # 分块读取（每次1024字节），适用于大二进制文件，避免内存占用过高
    while True:
        chunk = read_f.read(1024)  # 读取1024字节块
        if not chunk:  # 读取完毕（chunk为空字节串）
            break
        write_f.write(chunk)  # 写入字节块到备份文件
print("二进制文件备份完成")
```

### （2）writelines(iterable)：写入可迭代对象（列表/元组等）
- 接收字符串/字节串组成的**可迭代对象**，批量写入
- 不会自动添加换行符，需手动在元素中添加 \n
```python
# 基础示例：批量写入多行固定内容
lines = ["第一行\n", "第二行\n", "第三行\n"]
with open("test.txt", "w", encoding="utf-8") as f:
    f.writelines(lines)  # 批量写入列表中的字符串，\n 实现换行

# 实际应用1：读取多个文件内容，整合后批量写入新文件
# 场景：合并多个日志文件内容到一个汇总文件
log_files = ["log1.txt", "log2.txt", "log3.txt"]
summary_lines = []

# 先读取所有日志文件内容，添加到列表中（带文件标识）
for idx, log_file in enumerate(log_files, 1):
    with open(log_file, "r", encoding="utf-8") as f:
        # 给每个文件的内容添加标识行，便于区分来源
        summary_lines.append(f"=== 日志文件{idx}：{log_file} ===\n")
        summary_lines.extend([line for line in f])  # 读取文件所有行，添加到列表
        summary_lines.append("\n")  # 每个文件内容后添加空行分隔

# 批量写入汇总文件
with open("log_summary.txt", "w", encoding="utf-8") as f:
    f.writelines(summary_lines)
print("多日志文件合并完成")

# 实际应用2：结构化数据格式化批量写入（生成CSV文件）
# 场景：将字典列表格式的用户数据，转为CSV行后批量写入
user_list = [
    {"name": "张三", "age": 25, "city": "北京"},
    {"name": "李四", "age": 30, "city": "上海"},
    {"name": "王五", "age": 28, "city": "广州"}
]

# 先构造CSV表头和内容行（转为字符串列表）
csv_lines = ["姓名,年龄,城市\n"]  # 表头
for user in user_list:
    # 格式化每个用户数据为CSV行字符串
    csv_line = f"{user['name']},{user['age']},{user['city']}\n"
    csv_lines.append(csv_line)

# 批量写入CSV文件
with open("user_info.csv", "w", encoding="utf-8") as f:
    f.writelines(csv_lines)
print("用户数据CSV文件生成完成")
```

## 5.读写模式（r+、w+、a+）
若需同时实现读写操作，可使用带“+”的模式，核心区别如下：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111211850795.png)
**注意**：可读可写模式下，“读”、“写”共用一个“文件指针”，会相互影响

### （1）r+ 模式：读写（文件必须存在）
假设 test.txt 初始内容为：
```
Hello World
Python File
End of File
```
```python
# r+ 模式：从文件开头开始，可读可写
with open("test.txt", "r+", encoding="utf-8") as f:
    # 1. 先读取内容
    content = f.read()
    print(f"读取内容：\n{content}")
    print(f"读取后指针位置：{f.tell()}")  # 指针在文件末尾
    
    # 2. 直接写入（会追加到末尾，因为指针在末尾）
    f.write("\n新增内容1")
    
    # 3. 移动指针到开头再写入（会覆盖原内容）
    f.seek(0)
    f.write("COVER")  # 覆盖前5个字符 "Hello" -> "COVER"
    
    # 4. 读取最终内容
    f.seek(0)
    print(f"最终内容：\n{f.read()}")
```
输出：
```
读取内容：
Hello World
Python File
End of File
读取后指针位置：35
最终内容：
COVER World
Python File
End of File
新增内容1
```

### （2）w+ 模式：读写（清空文件/创建新文件）
假设 test.txt 初始内容为：
```
Hello World
Python File
End of File
```
```python
# w+ 模式：打开时清空文件，可读可写
with open("test.txt", "w+", encoding="utf-8") as f:
    # 1. 文件已被清空，直接读取为空
    content = f.read()
    print(f"打开后读取：'{content}'")  # 空字符串
    
    # 2. 写入新内容
    f.write("第一行\n")
    f.write("第二行\n")
    f.write("第三行")
    print(f"写入后指针位置：{f.tell()}")
    
    # 3. 必须移动指针才能读取
    f.seek(0)
    print(f"读取写入的内容：\n{f.read()}")
```
输出：
```
打开后读取：''
写入后指针位置：21
读取写入的内容：
第一行
第二行
第三行
```

### （3）a+ 模式：追加读写（不清空/创建新文件）
假设 test.txt 初始内容为：
```
Hello World
Python File
End of File
```
```python
# a+ 模式：写入永远在末尾，读取需要移动指针
with open("test.txt", "a+", encoding="utf-8") as f:
    # 1. 打开时指针在末尾
    print(f"打开时指针位置：{f.tell()}")
    
    # 2. 直接读取为空（指针在末尾）
    content = f.read()
    print(f"直接读取：'{content}'")  # 空
    
    # 3. 移动指针到开头再读取
    f.seek(0)
    print(f"移动指针后读取：\n{f.read()}")
    
    # 4. 写入内容（无论指针在哪，都追加到末尾）
    f.seek(0)  # 即使移动到开头
    f.write("\n追加的内容")  # 仍然写到末尾
    
    # 5. 验证
    f.seek(0)
    print(f"最终内容：\n{f.read()}")
```
输出：
```
打开时指针位置：35
直接读取：''
移动指针后读取：
Hello World
Python File
End of File
最终内容：
Hello World
Python File
End of File
追加的内容
```

### （4）指针影响演示
假设 test.txt 初始内容为：
```
Hello World
Python File
End of File
```
```python
with open("demo.txt", "w+", encoding="utf-8") as f:
    # 写入内容
    f.write("ABCDEFGHIJ")  # 指针在位置10
    
    # 不移动指针直接读取
    result1 = f.read()
    print(f"写入后直接读：'{result1}'")  # 空，因为指针在末尾
    
    # 移动到位置5
    f.seek(5)
    result2 = f.read()
    print(f"从位置5读取：'{result2}'")  # FGHIJ
    
    # 在位置3写入，会覆盖
    f.seek(3)
    f.write("XYZ")  # 覆盖 DEF -> XYZ
    
    # 读取全部
    f.seek(0)
    print(f"最终内容：'{f.read()}'")  # ABCXYZGHIJ
```
输出
```
写入后直接读：''
从位置5读取：'FGHIJ'
最终内容：'ABCXYZGHIJ'

```

### 补充1：seek() 函数
在读写模式下，文件指针的位置直接影响读写结果，**seek函数**用于手动移动文件指针，是实现精准读写的关键。
- **核心功能**：
	- 移动文件指针到指定位置，支持文本模式和二进制模式（二进制模式更精准，文本模式下 offset 建议为 0）
- **语法**：
	- **seek(offset, whence)** 
- **参数说明**：
	- **offset**：偏移量，即指针移动的字节数
		- 正数向右移动
		- 负数向左移动
		- 文本模式下不支持负数偏移
	- **whence**：基准位置，可选值为 0、1、2（**默认值为 0**）
		- **whence=0**：以文件开头为基准（最常用，支持所有模式）；
		- **whence=1**：以当前指针位置为基准（仅支持二进制模式）；
		- **whence=2**：以文件末尾为基准（仅支持二进制模式）。
- **常见用法**：
	- f.seek(0)：指针移到文件开头（读写模式下读取前必用）；
	- f.seek(0, 2)：指针移到文件末尾（追加内容前可用，等同于 a 模式）；
	- f.seek(5)：指针从文件开头向右移动 5 个字节（精准定位修改位置）。
- **注意：**文本模式下 seek() 的限制
	- whence=1 和 whence=2 仅支持二进制模式
	- 文本模式下 offset 必须为 0，或者是 tell() 返回的值
		- 否则会报错：io.UnsupportedOperation: can't do nonzero cur-relative seeks

### 补充2：tell() 函数
tell() 是文件对象的方法，用于**获取当前文件指针的位置**。
#### <1>介绍
- **基本语法**：
```python
position = f.tell()  # 返回当前指针位置（字节数）
```
- **核心特点**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111220721101.png)
- **与seek()的配合关系**
```python
tell() ──► 获取当前位置（只读）
seek() ──► 设置指针位置（可写）

常见组合：
1. pos = f.tell()   # 保存位置
2. f.read(...)      # 读取操作
3. f.seek(pos)      # 恢复位置
```

#### <2>使用示例
##### 1）基本用法
```python
with open("test.txt", "r", encoding="utf-8") as f:
    print(f.tell())      # 输出：0（刚打开，指针在开头）
    
    f.read(5)            # 读取 5 个字符
    print(f.tell())      # 输出：5（英文）或更大（含中文时）
    
    f.read()             # 读取剩余全部内容
    print(f.tell())      # 输出：文件总字节数（指针在末尾）
```
##### 2）配合 seek() 使用
```python
with open("test.txt", "r", encoding="utf-8") as f:
    f.read(10)
    pos = f.tell()       # 记录当前位置
    print(f"当前位置：{pos}")
    
    content = f.read(5)  # 继续读取
    
    f.seek(pos)          # 回到之前记录的位置
    print(f.tell())      # 输出：与之前的 pos 相同
```

##### 3）读写模式下跟踪指针
```python
with open("demo.txt", "w+", encoding="utf-8") as f:
    print(f"写入前：{f.tell()}")   # 输出：0
    
    f.write("Hello Python")
    print(f"写入后：{f.tell()}")   # 输出：12
    
    f.seek(0)
    print(f"seek后：{f.tell()}")   # 输出：0
    
    f.read(5)
    print(f"读取后：{f.tell()}")   # 输出：5
```

##### 4）中文字符的字节计算
```python
# UTF-8 编码下，中文字符占 3 个字节
with open("chinese.txt", "w", encoding="utf-8") as f:
    f.write("你好世界")

with open("chinese.txt", "r", encoding="utf-8") as f:
    f.read(1)            # 读取 1 个字符（"你"）
    print(f.tell())      # 输出：3（1个中文 = 3字节）
    
    f.read(1)            # 读取 1 个字符（"好"）
    print(f.tell())      # 输出：6
```

##### 5）使用场景
```python
# 场景1：记录读取进度（断点续读）
with open("large_file.txt", "r", encoding="utf-8") as f:
    for i, line in enumerate(f):
        if i == 100:
            checkpoint = f.tell()  # 记录第100行后的位置
            print(f"检查点位置：{checkpoint}")
            break

# 场景2：计算文件大小
with open("test.txt", "rb") as f:
    f.seek(0, 2)         # 移动到文件末尾
    file_size = f.tell() # 获取文件大小（字节）
    print(f"文件大小：{file_size} 字节")
```




## 6.结构化数据存储：JSON（持久化存储）
- 当需要存储列表、字典、类实例等结构化数据时，直接用文本文件读写会导致格式混乱，推荐使用 Python 标准库提供的序列化工具：json（通用格式，跨语言兼容）、pickle（Python 专属格式，支持更多数据类型）。
- JSON（JavaScript Object Notation）是轻量级数据交换格式
	- 支持**字符串**、**数字**、**列表**、**字典**等基础数据类型
	- 不支持 Python 专属类型（如**类实例**、**元组**、**集合**）
		- **补充**：JSON 中数组会被转为 Python 列表，Python 元组序列化后会变成列表。读取时无法还原为元组。
	- **json.dump()**：写入文件,实现序列化
	- **json.load()**：读取文件，实现反序列化

### （1）写入JSON数据（序列化）

#### <1>json.dump() 函数
```python
1. 核心作用：将 Python 结构化数据（如字典、列表）序列化（转换）为 JSON 格式字符串，直接写入文件
2. 函数语法：json.dump(obj, fp, ensure_ascii=True, indent=None, separators=None, sort_keys=False, ...)
3. 关键参数说明：
    - obj：必传参数，要序列化的 Python 数据（支持字典、列表、字符串、数字、布尔值、None）；
    - fp：必传参数，文件对象（需以文本模式 "w" 或 "a" 打开），用于接收 JSON 数据；
    - ensure_ascii：可选参数，默认 True，非 ASCII 字符（如中文）会转为 Unicode 转义符（\uXXX）；设为 False 可保留中文原样；
    - indent：可选参数，默认 None（无格式化），传入整数时表示缩进空格数，用于格式化 JSON 输出（便于阅读）；
    - sort_keys：可选参数，默认 False，设为 True 时会按字典的键（key）升序排序输出。
```

#### <2>使用示例
```python
# 实际示例：写入 JSON 数据
import json

# 要存储的结构化数据（字典）
user_data = {
    "name": "张三",
    "age": 25,
    "hobbies": ["篮球", "编程"],
    "is_student": False
}

# 写入 JSON 文件
with open("user_data.json", "w", encoding="utf-8") as f:
    # ensure_ascii=False 确保中文正常显示，indent=2 格式化输出（便于阅读）
    json.dump(user_data, f, ensure_ascii=False, indent=2)
```

### （2）读取 JSON 数据（反序列化）

#### <1>json.load()函数
```python
 1. 核心作用：从已打开的 JSON 文件中读取 JSON 格式字符串，反序列化为 Python 对应数据类型（字典/列表等）
 2. 函数语法：json.load(fp, cls=None, object_hook=None, parse_float=None, parse_int=None, parse_constant=None, object_pairs_hook=None, ...)
 3. 关键参数说明：
    - fp：必传参数，文件对象（需以文本模式 "r" 打开），用于读取 JSON 数据；
    - object_hook：可选参数，自定义反序列化规则，接收一个字典参数，返回处理后的对象（如将 JSON 字典转为自定义类实例）；
    - parse_float/parse_int：可选参数，自定义浮点数/整数的解析规则（如将浮点数转为 Decimal 类型，避免精度丢失）；
    - cls：可选参数，指定自定义的 JSON 解码器类，用于扩展反序列化功能；
    - 其他参数（如 object_pairs_hook）：用于控制 JSON 对象的解析顺序（如保留键的插入顺序）。
 4. 返回值特征：
    - 反序列化后的返回值类型与 JSON 数据结构对应：
    	- JSON 对象 → Python 字典
    	- JSON 数组 → Python 列表
    	- JSON 字符串 → Python 字符串
    	- JSON 数字 → Python 整数/浮点数
    	- JSON true/false/null → Python True/False/None。
```
#### <2>使用示例
```python
# 基础示例：读取 JSON 文件（反序列化）
import json

# 读取 JSON 文件（文件对象需以 "r" 模式打开，指定正确编码）
with open("user_data.json", "r", encoding="utf-8") as f:
    loaded_data = json.load(f)  # 反序列化为 Python 字典

# 使用读取的数据（验证返回值类型与数据可用性）
print(type(loaded_data))  # 输出：<class 'dict'>，验证返回值为字典类型
print(loaded_data["name"])  # 输出：张三（正常访问字典键值）
print(loaded_data["hobbies"][0])  # 输出：篮球（访问字典中的列表元素）

# 扩展示例：使用 object_hook 自定义反序列化（将 JSON 字典转为自定义类实例）
class User:
    def __init__(self, name, age, hobbies, is_student):
        self.name = name
        self.age = age
        self.hobbies = hobbies
        self.is_student = is_student
    
    def introduce(self):
        print(f"我是{self.name}，{self.age}岁，爱好{','.join(self.hobbies)}")

# 定义自定义反序列化函数，传入 object_hook
def json_to_user(dict_data):
    return User(dict_data["name"], dict_data["age"], dict_data["hobbies"], dict_data["is_student"])

# 读取 JSON 并转为 User 实例
with open("user_data.json", "r", encoding="utf-8") as f:
    user_instance = json.load(f, object_hook=json_to_user)

user_instance.introduce()  # 输出：我是张三，25岁，爱好篮球,编程
```

#### <3>补充示例
JSON 中数组会被转为 Python 列表，Python 元组序列化后会变成列表,读取时无法还原为元组
```python
import json

data = {"coords": (1, 2, 3)}  # 元组
json_str = json.dumps(data)
print(json_str)  # {"coords": [1, 2, 3]}  ← 变成了数组

loaded = json.loads(json_str)
print(type(loaded["coords"]))  # <class 'list'>  ← 无法还原为元组
```

## 7.结构化数据存储：Pickle（Python 专属场景）
Pickle 是 Python 专属序列化工具，支持几乎所有 Python 数据类型（包括类实例、元组、集合、函数等），但生成的文件只能被 Python 解析，不支持跨语言。核心通过 pickle.dump()（写入）和 pickle.load()（读取）实现。

### （1）写入 Pickle 数据（序列化）

#### <1>pickle.dump() 函数
```python
 1. 核心作用：将 Python 任意数据类型（如类实例、字典、列表、元组等）序列化（转换）为二进制字节流，直接写入文件
 2. 函数语法：pickle.dump(obj, file, protocol=None, fix_imports=True, buffer_callback=None)
 3. 关键参数说明：
    - obj：必传参数，要序列化的 Python 数据（支持几乎所有 Python 类型，包括类实例、函数、集合、元组等）；
    - file：必传参数，文件对象（需以二进制模式 "wb" 或 "ab" 打开），用于接收序列化后的二进制字节流；
    - protocol：可选参数，指定序列化协议版本（默认使用最高兼容版本），取值范围 0-5：
      - protocol=0：文本格式序列化（兼容性最强，速度慢，不推荐）；
      - protocol>=1：二进制格式序列化（推荐，速度快、体积小），版本越高功能越全，需注意 Python 版本兼容性；
    - fix_imports：可选参数，默认 True，用于在 Python 2 和 Python 3 之间兼容，处理导入路径差异；
    - buffer_callback：可选参数，用于处理大数据对象的缓冲，自定义字节流写入方式（进阶用法）。
 4. 核心特征：
    - 支持 Python 专属类型：这是与 json.dump() 的核心区别，可直接序列化类实例、函数等 JSON 不支持的类型；
    - 二进制输出：序列化后的数据是二进制字节流，需用二进制模式写入文件，生成的文件不可直接阅读；
    - Python 专属：生成的文件只能被 Python 解析，无法跨语言使用（如 Java、JavaScript 无法读取）。
```

#### <2>使用示例
```python
# 实际示例：写入 Pickle 数据（序列化类实例）
import pickle

# 定义一个类（Pickle 支持类实例序列化）
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def introduce(self):
        print(f"我是{self.name}，{self.age}岁")

# 创建类实例
person = Person("李四", 30)

# 写入 Pickle 文件（必须用二进制模式 "wb"，无需指定 encoding）
with open("person.pkl", "wb") as f:
    # 使用默认最高协议版本序列化，也可指定 protocol=4（Python 3.4+ 支持）
    pickle.dump(person, f)
```
### （2）读取 Pickle 数据（反序列化）

#### <1>pickle.load() 函数
```python
 1. 核心作用：从已打开的 Pickle 二进制文件中读取序列化的字节流，反序列化为原本的 Python 数据类型（如类实例、字典、列表等）
 2. 函数语法：pickle.load(file, fix_imports=True, encoding="ASCII", errors="strict", buffers=None)
 3. 关键参数说明：
    - file：必传参数，文件对象（需以二进制模式 "rb" 打开），用于读取序列化的二进制字节流；
    - fix_imports：可选参数，默认 True，用于解决 Python 2 和 Python 3 之间的导入兼容性问题，自动调整导入路径；
    - encoding：可选参数，默认 "ASCII"，指定反序列化时字符串的编码方式（针对 Python 2 序列化的文件）；
      若读取 Python 3 序列化的文件，无需修改此参数，使用默认值即可；
    - errors：可选参数，默认 "strict"，指定编码错误的处理方式（如 "ignore" 忽略错误、"replace" 替换错误字符）；
    - buffers：可选参数，进阶用法，用于处理大数据对象的缓冲，接收一个可迭代的缓冲对象。
 4. 返回值特征：
    - 反序列化后的返回值类型与序列化前的原始数据类型完全一致（核心优势）：
      如序列化前是类实例 → 反序列化后仍是该类实例；序列化前是字典 → 反序列化后仍是字典；
    - 无需手动转换类型，可直接使用原始数据的属性和方法（如类实例的成员方法）。
```
```python
# 基础示例：读取 Pickle 文件（反序列化为原始 Python 类型）
import pickle

# 读取 Pickle 文件（必须用二进制模式 "rb" 打开，无需指定 encoding）
with open("person.pkl", "rb") as f:
    loaded_person = pickle.load(f)  # 反序列化为原本的 Person 类实例

# 验证返回值类型与功能可用性
print(type(loaded_person))  # 输出：<class '__main__.Person'>，确认为 Person 类实例
loaded_person.introduce()  # 输出：我是李四，30岁（可直接调用类实例方法）

# 扩展示例1：读取复杂 Pickle 数据（字典嵌套列表、元组）
# 先准备一个复杂数据并序列化
complex_data = {
    "name": "张三",
    "scores": [90, 85, 95],
    "info": ("Python开发者", 25),
    "is_valid": True
}
with open("complex_data.pkl", "wb") as f:
    pickle.dump(complex_data, f)

# 反序列化读取
with open("complex_data.pkl", "rb") as f:
    loaded_complex = pickle.load(f)

print(type(loaded_complex))  # 输出：<class 'dict'>
print(loaded_complex["scores"][1])  # 输出：85（正常访问列表元素）
print(loaded_complex["info"][0])  # 输出：Python开发者（正常访问元组元素）

# 扩展示例2：处理 Python 2 序列化的 Pickle 文件（兼容性场景）
# 若读取的是 Python 2 生成的 .pkl 文件，可能需要指定 encoding 参数
try:
    with open("python2_data.pkl", "rb") as f:
        # 用 "latin-1" 编码兼容 Python 2 序列化的字符串
        loaded_py2_data = pickle.load(f, encoding="latin-1")
    print("Python 2 序列化文件读取成功")
except Exception as e:
    print(f"读取失败：{e}")
```

## 8.Python 文件操作函数速查表
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111221730929.png)


# 十一、异常处理
- 在 Python 编程中，异常是程序运行时发生的非预期错误（如语法错误、逻辑错误、资源访问失败等），会导致程序中断。
- 异常处理的核心目标是**捕获并处理这些错误**，保证程序优雅运行而非直接崩溃，同时便于定位问题。

## 1.异常的本质
- Python 中所有异常都是 **BaseException** 类的子类，形成了层级分明的异常体系
- 常见的异常类（如 TypeError、FileNotFoundError）均继承自 **Exception 类**（BaseException 的子类）
- BaseException 还包含 SystemExit（程序退出）、KeyboardInterrupt（用户中断，如 Ctrl+C）等特殊异常。

## 2.常见异常类型及触发场景
Python 内置了大量异常类，覆盖开发中常见的错误场景，以下是高频异常类型汇总：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260111225712830.png)
**注**：这些异常类我们可以直接创建，并[通过raise 语句主动抛出](#raise)

## 3.异常处理核心语法
Python 提供 **try-except-else-finally** 语句块实现异常处理，各部分分工明确，可灵活组合使用。

### （1）基础结构：try-except
最核心的异常捕获结构：
- **try 块**：包裹可能触发异常的代码
- **except 块**：捕获并处理指定异常。

#### <1>语法格式
```python
try:
    # 可能触发异常的代码块（被监控的代码）
    危险操作
except 异常类型1 [as 异常对象]:
    # 捕获到“异常类型1”时执行的处理逻辑
    处理方案1
except 异常类型2 [as 异常对象]:
    # 捕获到“异常类型2”时执行的处理逻辑
    处理方案2
...
```
- **as 异常对象**(可选):
	- 用于获取异常的详细信息（如错误原因），通过异常对象.args 或直接打印异常对象查看。
- **多 except 顺序**:
	- 异常类有层级关系时，**需先捕获子类异常，再捕获父类异常**（否则子类异常会被父类异常捕获，无法精准处理）
- **未捕获异常**:
	- 若 try 块**触发的异常**未被任何 except 捕获，程序会中断并抛出默认错误信息。

#### <2>使用示例
```python
# 示例1：捕获指定异常（除零错误）
try:
    result = 10 / 0  # 触发 ZeroDivisionError
except ZeroDivisionError as e:
    print(f"除零错误：{e}")  # 输出：除零错误：division by zero
    print(f"错误详情：{e.args}")  # 输出：错误详情：('division by zero',)

# 示例2：多异常捕获（类型不匹配 + 除零错误）
try:
    num = int(input("请输入数字："))
    result = 10 / num
except ValueError as e:
    print(f"输入错误：{e}")  # 输入非数字时触发
except ZeroDivisionError as e:
    print(f"除零错误：{e}")  # 输入0时触发

# 示例3：捕获异常并修复（用户输入场景，引导修正错误）
def get_valid_number(prompt):
    while True:  # 循环直到获取有效输入
        try:
            num = int(input(prompt))  # 尝试转换输入为整数
            if num == 0:
                # 主动检测除零风险，抛出自定义提示的异常
                raise ValueError("输入不能为0，否则会触发除零错误")
            return num  # 输入有效，返回结果
        except ValueError as e:
            # 捕获输入错误，提示用户并引导重新输入（修复异常）
            print(f"输入无效：{e}，请重新输入！")

# 调用函数获取有效数字，确保后续计算正常执行
try:
    num = get_valid_number("请输入一个非零整数：")
    result = 10 / num
    print(f"计算成功！10 ÷ {num} = {result}")
except Exception as e:
    print(f"程序异常：{e}")
```

### （2）扩展结构：try-except-else
当 try 块未触发任何异常时，执行 else 块代码（else 块需紧跟所有 except 块之后）。

#### <1>语法格式
```python
try:
    可能触发异常的代码
except 异常类型 as e:
    异常处理逻辑
else:
    # try 块无异常时执行
    正常执行的后续逻辑
```

#### <2>使用示例
```python
try:
    num = int(input("请输入数字："))
    result = 10 / num
except (ValueError, ZeroDivisionError) as e:  # 用元组捕获多个异常
    print(f"错误：{e}")
else:
    print(f"计算成功，结果：{result}")  # 无异常时执行
```

### （3）完整结构：try-except-else-finally
finally 块用于定义“无论是否触发异常，都必须执行”的代码（如资源释放、文件关闭、连接断开等），是资源管理的核心。

#### <1>语法格式
```python
try:
    可能触发异常的代码
except 异常类型 as e:
    异常处理逻辑
else:
    无异常时执行的逻辑
finally:
    # 无论是否异常，必执行
    资源释放/清理逻辑
```
- **关键特性**
	- **优先级最高！！**：即使 try、except、else 块中有 return、break 等语句，finally 块仍会先执行。
	- **适用场景**：文件关闭、数据库连接断开、锁释放等必须执行的清理操作（替代手动关闭，避免异常导致资源泄漏）

#### <2>使用示例
```python
# 示例：演示 finally 的执行特性（注意：实际文件操作推荐用 with 语句）
def read_config(filename):
    file = None
    try:
        file = open(filename, "r", encoding="utf-8")
        content = file.read()
        return content  # 即使这里 return，finally 仍会执行
    except FileNotFoundError as e:
        print(f"文件不存在：{e}")
        return None
    except UnicodeDecodeError as e:
        print(f"编码错误：{e}")
        return None
    except Exception as e:
        # 捕获其他未预料的异常，避免程序崩溃
        print(f"读取文件时发生未知错误：{e}")
        return None
    finally:
        # 关键：无论是否发生异常，finally 都会执行
        if file:
            file.close()
            print("finally：文件已关闭")

result = read_config("test.txt")
print(f"返回值：{result}")

# 输出顺序（以文件存在为例）：
# finally：文件已关闭
# 返回值：（文件内容或 None）

# ⚠️ 注意：
# 上述代码仅用于演示 finally 的"必执行"特性
# 实际文件操作强烈推荐使用 with 语句（自动管理资源，代码更简洁安全）：
# with open("test.txt", "r", encoding="utf-8") as f:
#     content = f.read()
```
**再次强调**：上述代码仅用于演示 finally 的"必执行"特性，**实际文件操作强烈推荐使用 with！！** 

## 4.raise 语句（主动抛出异常）<a id="raise"></a>
除了捕获程序运行时的自然异常，还可通过 raise 语句主动触发异常，用于自定义错误校验逻辑（如参数合法性检查）。

### （1）基本语法
```python
# 语法1：抛出指定类型的异常（无详细信息）
raise 异常类型

# 语法2：抛出异常并指定错误信息
raise 异常类型("错误描述")

# 语法3：重新抛出捕获的异常（保留异常原始信息）
try:
    危险操作
except 异常类型 as e:
    # 处理部分逻辑后，重新抛出异常
    raise e  # 或直接 raise（保留完整异常栈）
```

### （2）使用示例
```python
# 示例1：参数合法性校验（主动抛出 ValueError）
def register(name, age):
    if not isinstance(name, str) or len(name) == 0:
        # 主动抛出异常，说明参数错误
        raise ValueError("用户名必须是非空字符串")
    if not isinstance(age, int) or age < 0 or age > 150:
        raise ValueError("年龄必须是 0-150 之间的整数")
    print(f"注册成功：姓名={name}，年龄={age}")

# 调用函数，触发主动抛出的异常
try:
    register("", 25)  # 用户名空，触发异常
except ValueError as e:
    print(f"注册失败：{e}")  # 输出：注册失败：用户名必须是非空字符串

# 示例2：重新抛出异常（保留原始错误栈）
try:
    register("张三", 200)
except ValueError as e:
    print(f"校验失败：{e}")
    raise  # 重新抛出异常，程序会中断并显示完整错误栈
```

## 5.自定义异常类
当 Python 内置异常无法满足业务需求时（如自定义业务错误类型），可**通过继承 Exception 类（或其子类）创建自定义异常**。

### （1）语法规则
- 需继承 Exception 类或其合法子类（如 ValueError、TypeError 等）
- 不建议直接继承 BaseException（避免捕获系统级异常，如KeyboardInterrupt）
- **通常需重写__init__ 方法**：可添加自定义属性（如错误码、详细描述），让异常信息更完整（可选，简单场景可直接使用父类构造方法）。

### （2）使用示例
```python
# ==================== 自定义异常类示例 ====================

# --- 示例1：最简单的自定义异常（直接继承，无额外属性）---
class MyBusinessError(Exception):
   """业务逻辑错误的基类"""
   pass


# --- 示例2：带自定义属性的异常（重写 __init__）---
class ValidationError(Exception):
   """数据验证错误"""
   def __init__(self, field, message, error_code=None):
       self.field = field          # 出错的字段名
       self.message = message      # 错误描述
       self.error_code = error_code  # 错误码（可选）
       # 调用父类构造方法，设置异常的默认消息
       super().__init__(f"[{error_code}] {field}: {message}")


# --- 示例3：继承特定异常子类 ---
class AgeValueError(ValueError):
   """年龄值错误（继承 ValueError，语义更明确）"""
   def __init__(self, age, message="年龄必须在 0-150 之间"):
       self.age = age
       self.message = message
       super().__init__(f"{message}，当前值：{age}")


# ==================== 使用示例 ====================

def register_user(name, age, email):
   """用户注册函数（演示自定义异常的使用）"""
   # 验证姓名
   if not name or len(name) < 2:
       raise ValidationError("name", "姓名长度不能少于2个字符", "E1001")
   
   # 验证年龄
   if not isinstance(age, int) or age < 0 or age > 150:
       raise AgeValueError(age)
   
   # 验证邮箱
   if "@" not in email:
       raise ValidationError("email", "邮箱格式不正确", "E1002")
   
   return {"name": name, "age": age, "email": email}


# ==================== 捕获自定义异常 ====================

try:
   user = register_user("张", 25, "test@example.com")
except ValidationError as e:
   print(f"验证失败 - 字段：{e.field}，原因：{e.message}，错误码：{e.error_code}")
except AgeValueError as e:
   print(f"年龄错误 - {e.message}，输入值：{e.age}")
else:
   print(f"注册成功：{user}")

# 输出：验证失败 - 字段：name，原因：姓名长度不能少于2个字符，错误码：E1001

```

# 十二、代码测试

## 1.测试的分类

| 测试类型 | 说明 | 粒度 |
|----------|------|------|
| **单元测试** | 测试最小代码单元（函数/方法） | 最细 |
| **集成测试** | 测试模块间的协作 | 中等 |
| **端到端测试** | 测试完整业务流程 | 最粗 |

## 2.常用测试框架

### （1）unittest（标准库，内置）

```python
import unittest

def add(a, b):
    return a + b

class TestAdd(unittest.TestCase):
    def test_add_positive(self):
        self.assertEqual(add(1, 2), 3)
    
    def test_add_negative(self):
        self.assertEqual(add(-1, -2), -3)
    
    def test_add_zero(self):
        self.assertEqual(add(0, 0), 0)

# 这个判断的作用是"当直接运行该测试文件时执行测试，被导入时不执行"
if __name__ == "__main__":
    unittest.main()
```

**常用断言方法**：

| 方法 | 作用 |
|------|------|
| `assertEqual(a, b)` | a == b |
| `assertNotEqual(a, b)` | a != b |
| `assertTrue(x)` | x 为真 |
| `assertFalse(x)` | x 为假 |
| `assertIsNone(x)` | x is None |
| `assertRaises(Error)` | 检查是否抛出指定异常 |


### （2）pytest（第三方，更推荐 ⭐）

```bash
pip install pytest
```

```python
# test_demo.py
def add(a, b):
    return a + b

# 函数名以 test_ 开头即可，无需继承类
def test_add_positive():
    assert add(1, 2) == 3

def test_add_negative():
    assert add(-1, -2) == -3

# 测试异常
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("除数不能为0")
    return a / b

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)
```

**运行测试**：
```bash
pytest                    # 自动发现并运行所有 test_*.py
pytest -v                 # 详细输出
pytest test_demo.py       # 运行指定文件
pytest -k "add"           # 只运行名称包含 "add" 的测试
```

### （3）pytest 参数化测试（批量测试多组数据）

```python
import pytest

@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (-1, -2, -3),
    (0, 0, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    assert add(a, b) == expected
```

## 3.测试夹具（Fixture）—— 测试前后的准备/清理

### <1>unittest 方式

```python
class TestDatabase(unittest.TestCase):
    def setUp(self):
        """每个测试方法前执行"""
        self.db = connect_database()
    
    def tearDown(self):
        """每个测试方法后执行"""
        self.db.close()
    
    def test_query(self):
        result = self.db.query("SELECT 1")
        self.assertIsNotNone(result)
```

### <2>pytest 方式（更灵活）

```python
import pytest

@pytest.fixture
def db_connection():
    """夹具：建立数据库连接"""
    conn = connect_database()
    yield conn  # yield 之前是 setup，之后是 teardown
    conn.close()

def test_query(db_connection):  # 参数名 = 夹具函数名
    result = db_connection.query("SELECT 1")
    assert result is not None
```

## 4.Mock 模拟（隔离外部依赖）

测试时模拟网络请求、数据库、文件等外部依赖，避免真实调用。

```python
from unittest.mock import Mock, patch
import requests

# 示例：模拟 API 请求
def get_user_name(user_id):
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()["name"]

# 测试时模拟 requests.get
@patch("requests.get")
def test_get_user_name(mock_get):
    # 设置模拟返回值
    mock_get.return_value.json.return_value = {"name": "张三"}
    
    result = get_user_name(123)
    
    assert result == "张三"
    mock_get.assert_called_once()  # 验证被调用过
```

## 5.测试覆盖率

```bash
pip install pytest-cov
pytest --cov=your_module --cov-report=html
```
生成覆盖率报告，查看哪些代码未被测试覆盖。


## 6.实际开发中的测试最佳实践

| 原则 | 说明 |
|------|------|
| **测试文件命名** | `test_*.py` 或 `*_test.py` |
| **测试函数命名** | `test_功能_场景_预期结果` |
| **一个测试一个断言** | 每个测试只验证一个行为 |
| **独立性** | 测试之间互不依赖，可单独运行 |
| **快速** | 单元测试应毫秒级完成 |
| **使用 Mock** | 隔离外部依赖（网络、数据库、文件） |

## 7.项目结构示例

```text
my_project/
├── src/
│   └── calculator.py
├── tests/
│   ├── __init__.py
│   ├── test_calculator.py
│   └── conftest.py        # pytest 共享夹具
├── pytest.ini             # pytest 配置
└── requirements.txt
```

## 8.快速对比

| 特性 | unittest | pytest |
|------|----------|--------|
| 安装 | 内置 | 需安装 |
| 语法 | 类 + 方法 | 函数即可 |
| 断言 | `self.assertEqual()` | 直接 `assert` |
| 参数化 | 繁琐 | `@pytest.mark.parametrize` |
| 夹具 | `setUp/tearDown` | `@pytest.fixture` |
| **推荐度** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |


**总结**：实际开发中 **pytest 是主流选择**，语法简洁、功能强大、插件丰富。结合 Mock 隔离依赖 + 覆盖率检测，可构建可靠的测试体系。

