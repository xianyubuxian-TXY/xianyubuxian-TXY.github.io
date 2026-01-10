---
title: 第一章：Python数据模型
date: 2026-01-09 12:14:40
tags:
	- 笔记
categories:
	- Python
	- 《流畅的python》
	- 第1章 Python数据模型
---

# 三、特殊方法概述
**之所以把这放在最开始，是为了方便熟悉以后查找特殊方法**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109155341846.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109155359478.png)


# 一、概述
## 1.特殊方法（"魔术方法"）
- **特征**：名称前后两端都有双下划线。如：
	- 在 obj[key] 句法背后提供支持的是特殊方法 __getitem__

- **作用**：在特殊语句中自动触发调用，不需要显示手动调用。如：
	- my_collection[key]：Python 解释器自动调用my_collection.__getitem__(key)。
	- print(my_object)：Python 解释器自动调用print(my_object.__repr__())
**很大程度上方便了我们的使用**
- **重要性**：想让对象支持以下基本的语言结构并与其交互，就需要实现特殊方
法：
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109141629741.png)

## 2.实例：“纸牌”
### (1) 实现+“特殊方法”
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109142035218.png)
- **Card**： 纸牌类，只有属性无方法
	-  **collections.namedtuple('类名', ['字段1', '字段2', ...])**：用于构建只有属性而没有自定义方法的类对象
- **FrenchDeck**： 法式扑克牌类，包含Card列表、**魔法方法**
	- [str(n) for n in range(2, 11)] ：“**列表推导式语法**”，下一章会学到
	- **魔法方法**：\_\_getitem\_\_ 和 \_\_len\_\_ 

**本节关键：FrenchDeck类**

### (2)“FrenchDeck类”的使用
#### <1>返回一摞牌有多少张
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109143542901.png)
- **得益于\_\_len\_\_魔法方法**：底层调用了该魔法方法

#### <2>从这摞牌中抽取某一张
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109143739371.png)
- **得益于\_\_getitem\_\_魔法方法**：底层调用了该魔法方法


#### <3>“\_\_getitem\_\_魔法方法” 其它功效
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109143955133.png)
- **支持“切片”**

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109144028869.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109144054149.png)
- **支持“正向/反向 迭代”**

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109144216884.png)
- **迭代往往是隐式的**：如果一个容器没有实现 __contains__ 方法，那么 in 运算符就会做一次顺序扫描
	- FrenchDeck 类支持 in 运算符，因为该类可迭代。

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109144536685.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109144549525.png)
- **支持“排序”**

1.通过前面使用reversed 和 sorted 的示例可以看出，实现 \_\_len\_\_ 和 \_\_getitem\_\_ 两个特殊方法后，FrenchDeck 的行为就像标准的 Python序列一样，受益于语言核心特性（例如迭代和切片）和标准库。
2.\_\_len\_\_ 和 \_\_getitem\_\_ 的实现利用组合模式，把所有工作委托给一个 list 对象，即 self.\_cards.

# 二、特殊方法是如何使用的
## 1.概述
- **特殊方法供 Python 解释器调用，而不是你自己**
	- 如：没有 my_object.\_\_len\_\_() 这种写法，正确的写法是len(my_object)
- **如果my_object 是用户定义的类的实例，Python将调用你实现的“魔法方法”**
	- 如：\_\_len\_\_方法
- **很多时候，特殊方法是“隐式调用”的！**
	- 如：for i in x: 语句其实在背后调用 iter(x)，接着又调用x.\_\_iter\_\_()（前提是有该方法）或 x.\_\_getitem\_\_()
- **特殊方法的调用**
	- 在编写代码时**一般不直接显示调用**特殊方法，除非涉及大量元编程。**唯一例外的是\_\_init\_\_ 方法**，为自定义的类实现 \_\_init\_\_ 方法时经常直接调用它调取超类的初始化方法。
	- 如果需要调用特殊方法，则**最好调用相应的内置函数**，例如len、iter、str等。这些内置函数不仅调用对应的特殊方法，通常还提供额外服务，而且对于内置类型来说，速度比调用方法更快

## 2.实例：Vector类
**Vector类模拟一个二维向量类，即数学和物理中使用的欧几里得向量**
### （1）Vector类的实现
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109150648100.png)
- **\_\_abs\_\_：绝对值**
- **\_\_add\_\_、\_\_mul\_\_：分别用于实现 + 和 * 运算符**

**使用：**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109151219492.png)

### （2） Vector 类的其他特殊方法
#### <1>字符串表示形式
##### 1.\_\_repr\_\_ 特殊方法
- **供内置函数 repr 调用**：获取对象的**标准**字符串表示形式。
	- 如未定义 __repr__ 方法，Vector 实例在 Python 控制台中显示为 <Vector object at 0x10e100070> 形式
- **__repr__ 方法返回的字符串应当没有歧义**：最好与源码保持一致，方便重新创建所表示的对象。
	- **!r**:   Vector 类 \_\_repr\_\_ 方法中的 f 字符串使用 !r 以**标准的表示形式显示属性**。
		- 这样做比较好，因为 Vector(1, 2) 和Vector('1', '2')之间是有区别的，后者在这个示例中不可用，因为构造函数接受的参数是数值而不是字符串。

##### 2.\_\_str\_\_ 特殊方法
- **供内置函数 str() 调用**：返回**对终端用户友好**的字符串。
- 有时，__repr__ 方法返回的字符串足够友好，无须再定义 \_\_str\_\_方法，因为继承自 object 类的实现最终会调用 \_\_repr\_\_ 方法

##### 3.\_\_repr\_\_ 与 \_\_str\_\_ 的区别
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109153141165.png)

#### <2>自定义类型的布尔值
- **Python 有一个 bool 类型，在需要布尔值的地方处理对象**
	- 如：if 或 while 语句的条件表达式，或者 and、or 和 not 的运算对象。
- **为了确定 x 表示的值为 "真" 或为 "假"，Python 调用 bool(x)，返回 True 或 False。**
- **默认情况下，用户定义类的实例都是真值，除非实现了 \_\_bool\_\_ 或 \_\_len\_\_ 方法**
	- **实现了\_\_bool\_\_ 方法**：bool(x) 调用 x.\_\_bool\_\_()，以后者返回的结果为准。
	- **没有实现 \_\_bool\_\_ 方法**：则 Python 尝试调用x.\_\_len\_\_()；
		- 如果该方法返回零值，则 bool 函数返回 False，否则返回 True

#### <3>补充：Python中，万物皆对象 ——> 万物皆有布尔值
- **未实现\_\_bool\_\_**：布尔值恒为True
- **实现了\_\_bool\_\_**：以x.\_\_bool\_\_()调用结果为准
	- **空容器、0、None、False被视为假**
	- **其他值被视为真** 

#### <4>自定义API （先了解，可以之后再回头看）
**本节简要说明 Python 中最重要的容器接口，纵览容器类型对特殊方法
的使用情况。**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109154613470.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260109154741486.png)
