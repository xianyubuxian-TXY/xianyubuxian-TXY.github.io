---
title: C++宏的使用
date: 2026-01-27 14:38:12
tags:
	- 笔记
categories:
	- C++
	- 开发笔记
	- C++宏的使用
---

# 一、宏的介绍
- 宏是 C/C++ **预处理阶段（编译前）**由预处理器处理的**文本替换指令**
	- 不涉及类型检查，也不属于 C++ 语言的语法范畴
- 宏通过**`#define`**定义，**`#undef`**取消定义，预处理阶段完成替换后，宏本身会从代码中消失。
- **核心**：
	- **文本替换**

**宏的常用符号表**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260127145716761.png)

# 二、宏的分类与基本用法
- **本质**：“文本替换”

## 1.无参宏（常量宏）
- 最基础的宏，用于定义常量或通用文本片段，替换时**直接替换字符串**。
```cpp
#include <iostream>
using namespace std;

// 定义无参宏（常量）
#define PI 3.1415926
#define MAX_BUFFER_SIZE 1024
#define HELLO "Hello, Macro!"

int main() {
    double r = 5.0;
    // 预处理后：double area = 3.1415926 * 5.0 * 5.0;
    double area = PI * r * r;
    cout << "面积：" << area << endl;
    cout << HELLO << endl; // 替换为cout << "Hello, Macro!" << endl;
    
    // 取消宏定义（后续代码中PI不再生效）
    #undef PI
    // 下面这行会报错，因为PI已被取消定义
    // double circle = 2 * PI * r;
    return 0;
}
```
**注意**：
- 末尾**不要加分号**，否则替换后可能出现语法错误（如#define PI 3.14;，替换后area = 3.14; * r * r;，直接报错）。

## 2.带参宏（函数宏）
- 模拟函数的宏，支持参数，本质是 “**带参数的文本替换**”，比函数调用更高效（无函数调用开销），但风险更高。

```cpp
#include <iostream>
using namespace std;

// 定义带参宏：求两数最大值
// 注意：参数加括号、整体加括号，避免运算符优先级问题
#define MAX(a, b) ((a) > (b) ? (a) : (b))
// 定义带参宏：交换两个数
#define SWAP(a, b) { auto temp = a; a = b; b = temp; }

int main() {
    int x = 10, y = 20;
    // 预处理后：int max_val = ((10) > (20) ? (10) : (20));
    int max_val = MAX(x, y);
    cout << "最大值：" << max_val << endl; // 输出20
    
    SWAP(x, y); // 替换为{ auto temp = x; x = y; y = temp; }
    cout << "交换后x：" << x << "，y：" << y << endl; // 输出x=20，y=10
    
    return 0;
}
```
**关键注意事项**：
- **参数**、**整体**必须都加括号！否则会因运算符优先级出错：
	- **错误示例**：`#define MUL(a,b) a*b`，调用`MUL(2+3,4)`会替换为`2+3\*4`，结果是 14 而非 20；
	- 正确写法：`#define MUL(a,b) ((a)*(b))`，替换后`((2+3)*(4))`，结果正确。
- 带参宏**不检查参数类型**（因为只是文本替换）：
	- MAX(3.5, 2)和MAX("a", "b")都会被替换（后者可能运行时出错，但编译预处理阶段不报错）。

## 3.多行宏
- 当宏内容超过一行时，用`\`（反斜杠）换行（末尾无空格），最后一行不需要`\`。
```cpp
#include <iostream>
using namespace std;

// 多行宏：打印变量名和值
#define PRINT_VAR(var) \
    cout << "变量名：" << #var << endl; \
    cout << "变量值：" << var << endl;

int main() {
    int num = 100;
    // 预处理后会展开为两行cout语句
    PRINT_VAR(num);
    return 0;
}
```
**输出**：
```bash
变量名：num
变量值：100
```

# 三、宏的高阶用法
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260127145716761.png)

- 宏定义**很难直接写**出，一般都是**先写出C++不同代码**，**在对照着写**对应的宏定义
- **推荐**：
	- **还是让AI写吧，但可以熟悉一下**
	- **只要你认为你的代码某些地方结构很类似，感觉代码重复了，就可以考虑用“宏定义”来减少代码量**
- **缺点**：
	- **“宏定义”不好debug，出问题难排查**

## 1.“宏定义”在enum上的使用（减少重复代码）
### （1）普通的enum的定义
- 每一个enum都需要重新定义`ErrorCodeToString`、`StringToErrorCode`,代码量太大
```cpp

enum class ErrorCode{
	OK 					= 0,	// 成功	
    Unknown             = 100,  // 未知错误
    Internal            = 101,  // 内部服务器异常
    ......
};

// Error -> string
inline std::string ErrorCodeToString(ErrorCode code){
	switch (code){
		case ErrorCode::Ok:                  return "Ok";
        case ErrorCode::Unknown:             return "Unknown";
        case ErrorCode::Internal:            return "Internal";
        ......
		default:                             return "Unknown";
	}
}

// string -> Error
inline ErrorCode StringToErrorCode(const std::string str){
	static std::unordered_map<std::string,ErrorCode> enum_map={
		{"Ok",ErrorCode::Ok},
		{"Unknown",ErrorCode::Unknown},
		{"Internal",ErrorCode::Internal}
		......
	};
	auto it=enum_map.find(str);
	if(it!=enum_map.end()){
		return it->second;
	}
	throw std::invalid_argument("Unknown ErrorCode, value： " + str);
}
```

### （2）宏定义的enum
- 注意，一般都是先写出“不同的enum的定义”，然后对照着来写“宏定义的enum”
- **关键**：找到相同的结构部分，抽取出“统一结构”，并用“占位符”先代替，“占位符”的具体值有使用者定义（毕竟要哪些枚举值由使用者决定）
**宏定义**
```cpp
//enum_utils.h
#pragma once
#include <string>
#include <unordered_map>

// ============================================================
//                    枚举自动生成宏
// 
// 核心思想：
//   - ENUM_ENTRIES 是"占位符"，由用户定义具体枚举项
//   - X 是"统一结构"，用户在调用前重新定义来控制展开方式
// ============================================================

// 1.定义枚举类型：
#define DEFINE_ENUM(EnumName)												\
	enum class EnumName{													\
		/* 这种统一结构，都先用一个“占位符”，之后由使用者定义具体的“占位符” */	\
		ENUM_ENTITIES														\
	};



// 2.生成EnumToString函数：
#define DEFINE_ENUM_TO_STRING(EnumName)								\
	inline std::string EnumName##ToString(EnumName value){			\
		switch (value){												\
			/* “占位符”的值可能不同，具体要看如何定义这个“占位符” */	 	\
			ENUM_ENTITIES											\
			default: return "Unknown";								\
		}															\
	}	



// 3.生成StringToEnum函数：
#define DEFINE_STRING_TO_ENUM(EnumName)	\
	inline EnumName StringTo##EnumName(const std::string str){						\
		static std::unordered_map<std::string,EnumName> enum_map={				\
			/* 统一结构，先用“占位符”，之后由使用者定义具体的“占位符” */				\
			ENUM_ENTITIES														\
		}; 																		\
		auto it=enum_map.find(str);												\
		if(it!=enum_map.end()){													\
			return it->second;													\
		}																		\
		throw std::invalid_argument("Unknown " #EnumName ", value： " + str);	\
	};
```
**用户使用**
- 先明确：宏定义除非undef，否则是全局的，所以“用户定义的占位符”在预处理时会对“宏定义中的占位符”进行替换
```cpp
//error_code.h

// Step 1: 定义“占位符” —— 用下面的方式只要写一次即可
#define ENUM_ENTITIES	\
	X(Ok,       0)		\
	X(Unknown,  100)	\
	X(Internal, 101)

// 展开为：
// enum class ErrorCode {
//     Ok = 0,
//     Unknown = 100,
//     Internal = 101,
// };


// Step 2: 定义 X 的"统一结构" → 生成枚举
#undef X   							// 清理X宏，避免污染
#define X(name,value) name=value,	// 统一结构

DEFINE_ENUM(ErrorCode)				// 调用宏定义：生成枚举


// Step 3: 重新定义 X → 生成 ToString
/*	- 虽然在上面我们的统一结构中，X(name)就可以了，但为了与Setp 1 的统一，
	  这里也用X(name,value),value不会用到
	*/
#undef X 													// 清理X宏，避免污染		
#define X(name,value) case ErrorCode::name:  return #name;	// 统一结构

DEFINE_ENUM_TO_STRING(ErrorCode)							// 调用宏定义：生成ToString函数


// Step 4: 重新定义 X → 生成 StringToEnum （这里的value也单纯为了统一结构）
#undef X
#define X(name,value) {#name,ErrorCode::name},	
DEFINE_STRING_TO_ENUM(ErrorCode)


// 清理宏，避免污染
#undef X
#undef ENUM_ENTRIES
```
