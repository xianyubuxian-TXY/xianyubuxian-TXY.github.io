---
title: '[[noreturn]]属性'
date: 2026-01-26 20:04:20
tags:
	- 笔记
categories:
    - C++
    - C++特性
    - C++11特性
    - '[[noreturn]]属性'
---

# 1.介绍
`[[noreturn]]` 是 C++11 引入的标准属性，它表明函数在执行完毕后**不会返回到调用者**。
```cpp
[[noreturn]] void terminate_program() {
    std::exit(1);
}
```

# 2.“不返回”的含义
-  "不返回"不是指返回 void，而是指**控制流永远不会回到调用点。**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126202028378.png)

## （1）示例
### <1>不使用[[noreturn]]
- 这段代码可能看着有些怪，主要是为了方便演示“不返回”的含义
```cpp
// 没有 [[noreturn]] 时
void error_exit(const char* msg) {
    std::cerr << msg << std::endl;
    std::exit(1);
}

void success(const char* msg){
	std::cout<< msg << std::endl;
}

int calculate(int x) {
	if(x >= 0){
		success("valid value");
	}
    
	if(x >= 0){
		return x * 2;
	}

    if (x < 0) {
        error_exit("invalid value");
        // ⚠️ 警告：控制流可能到达非 void 函数末尾而不返回值
    }

}
// ⚠️ 编译器警告：控制流可能到达非 void 函数末尾而不返回值
// 编译器不知道 error_exit 不会返回，认为 if 分支没有 return
```
- **`x>0`时**：
	- 会先执行success函数，执行完后，会**返回到calculate函数中**，然后执行`return x*2`
- **`x<0`时**：
	- 会执行error_exit函数，内部会执行std::exit函数，程序会直接终止，也就**不会返回到calculate函数中**
	- 但编译其在编译calculate函数时，并不知道error_exit不会返回到calculate中，认为该情况下没有返回值，会**发出警告**

### <2>使用[[noreturn]]
```cpp
[[noreturn]] void error_exit(const char* msg) {
    std::cerr << msg << std::endl;
    std::exit(1);
}

void success(const char* msg){
	std::cout<< msg << std::endl;
}

int calculate(int x) {
	if(x >= 0){
		success("valid value");
	}
    
	if(x >= 0){
		return x * 2;
	}

    if (x < 0) {
        error_exit("invalid value");
        // ✅ 无警告：编译器知道这里不会返回
    }

}
```

## （2）标准库中的 [[noreturn]] 函数
```cpp
// 这些标准库函数都是 [[noreturn]]
std::abort();
std::exit(int);
std::quick_exit(int);
std::terminate();
std::rethrow_exception(std::exception_ptr);
std::unreachable();     // C++23 新增 ⭐
```

# 3.用途
## （1）消除编译器警告
```cpp
void ThrowError(const std::string& msg) {
    throw std::runtime_error(msg);
}

int GetValue(bool condition) {
    if (condition) {
        return 42;
    }
    ThrowError("condition is false");  // 没有 [[noreturn]]，编译器会警告"不是所有路径都有返回值"
}
```
```cpp
// ✅ 正确使用：函数抛出异常，永不返回
[[noreturn]] void ThrowError(const std::string& msg) {
    throw std::runtime_error(msg);
}

int GetValue(bool condition) {
    if (condition) {
        return 42;
    }
    ThrowError("condition is false");  // ✅ 无警告：编译器知道这里不会返回
}
```

## （2）编译器优化
### <1>优化场景示例
```cpp
int GetUserAge(int user_id) {
    auto user = FindUser(user_id);
    
    if (!user) {
        ThrowMySQLException(1001, "User not found");  // [[noreturn]]
    }
    
    return user->age;  // ← 编译器如何处理这里？
}
```

### <2>编译器的优化
#### 1）消除"无法到达"的代码路径分析
```cpp
没有 [[noreturn]] 时，编译器的理解：
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   if (!user) {                                              │
│       ThrowMySQLException(...);                             │
│       // 编译器认为：函数可能返回，需要继续执行              │
│       // 所以会生成"函数返回后的处理代码"                    │
│   }                                                         │
│   return user->age;  // 需要检查所有路径是否都有返回值       │
│                                                             │
└─────────────────────────────────────────────────────────────┘

有 [[noreturn]] 时，编译器的理解：
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   if (!user) {                                              │
│       ThrowMySQLException(...);                             │
│       // 编译器知道：这里永远不会返回                        │
│       // 不需要生成后续代码，直接跳过                        │
│   }                                                         │
│   return user->age;  // 编译器知道：到这里时 user 一定有效   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 2）汇编层面的优化
```cpp
// 源码
int Process(int x) {
    if (x < 0) {
        FatalError("negative value");  // [[noreturn]]
    }
    return x * 2;
}
```
```bash
; 没有 [[noreturn]] 的汇编（伪代码）
Process:
    cmp     edi, 0
    jl      .negative_branch
    ; 正常路径
    lea     eax, [rdi + rdi]
    ret
.negative_branch:
    call    FatalError
    ; ❌ 编译器可能生成：处理 FatalError 返回后的代码
    ; 例如：跳回主流程、清理栈帧等
    jmp     .after_if          ; 多余的跳转指令
.after_if:
    lea     eax, [rdi + rdi]
    ret

; 有 [[noreturn]] 的汇编（伪代码）
Process:
    cmp     edi, 0
    jl      .negative_branch
    lea     eax, [rdi + rdi]   ; ✅ 直接计算，不需要跳转
    ret
.negative_branch:
    call    FatalError
    ; ✅ 不生成任何后续代码，因为知道不会返回
    ; 甚至可能用 jmp 代替 call（尾调用优化）
```

#### 3）编译器优化列表
```cpp
┌─────────────────────────────────────────────────────────────┐
│                    编译器优化列表                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1️⃣ 删除死代码（Dead Code Elimination）                     │
│     • [[noreturn]] 函数调用后的代码直接删除                 │
│                                                             │
│  2️⃣ 简化控制流（Control Flow Simplification）              │
│     • 不需要生成"函数返回后跳转回来"的指令                  │
│     • 减少分支跳转                                          │
│                                                             │
│  3️⃣ 寄存器分配优化                                         │
│     • 不需要在 [[noreturn]] 调用前保存寄存器                │
│     • 因为不会返回，不需要恢复现场                          │
│                                                             │
│  4️⃣ 栈帧优化                                               │
│     • 某些情况下可以使用尾调用优化（tail call）             │
│     • 直接 jmp 而不是 call                                  │
│                                                             │
│  5️⃣ 消除警告 + 类型推断                                    │
│     • 编译器知道某些变量在特定路径后"一定有效"              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## （3）传达程序员意图，提高代码可读性
```cpp
// 一眼就能看出这个函数的行为
[[noreturn]] void fatal(const std::string& msg);
[[noreturn]] void throw_not_implemented();
[[noreturn]] void infinite_loop();
```

# 4.[[noreturn]] 实际使用场景
## （1）抛出异常的工厂函数
### <1>统一异常类型和格式
```cpp
// ✅ 推荐：整个项目统一异常风格
class ErrorThrower {
public:
    [[noreturn]] static void invalid_argument(const std::string& param, const std::string& reason) {
        throw std::invalid_argument("Invalid argument '" + param + "': " + reason);
    }
    
    [[noreturn]] static void out_of_range(size_t index, size_t size) {
        throw std::out_of_range(
            "Index " + std::to_string(index) + 
            " >= size " + std::to_string(size));
    }
    
    [[noreturn]] static void not_implemented(const std::string& feature) {
        throw std::logic_error("Not implemented: " + feature);
    }
};
```

### <2>需要附加操作（日志、指标等）
```cpp
// ✅ 推荐：抛异常前做额外工作
[[noreturn]] void throw_with_logging(const std::exception& e) {
    Logger::error("Exception: {}", e.what());
    Metrics::increment("exception_count");
    throw;  // 重新抛出
}
```

### <3>异常构造逻辑复杂时
```cpp
// ✅ 推荐：复杂的异常信息构造
[[noreturn]] void throw_parse_error(
    const std::string& filename, 
    int line, 
    int column,
    const std::string& expected,
    const std::string& got) 
{
    std::ostringstream oss;
    oss << filename << ":" << line << ":" << column 
        << " Parse error: expected '" << expected 
        << "', got '" << got << "'";
    throw ParseException(oss.str());
}

// 使用处简洁
void parse(const Token& tok) {
    if (tok.type != TokenType::Number) {
        throw_parse_error(file_, line_, col_, "number", tok.value);
    }
}
```

## （2）服务主循环
```cpp
class Server {
public:
    [[noreturn]] void run() {
        while (true) {
            auto conn = accept_connection();
            handle_request(conn);
        }
        // 永远不会到这里
    }
};

int main() {
    Server server;
    server.run();
    // ✅ 编译器知道 run() 不会返回
    // 后面的代码不会被执行
}
```

## （3）错误日志 + 程序终止
```cpp
enum class ErrorCode { OutOfMemory, FileNotFound, NetworkError };

[[noreturn]] void fatal_error(ErrorCode code, const std::string& detail) {
    std::cerr << "Fatal error [" << static_cast<int>(code) << "]: " 
              << detail << std::endl;
    
    // 可以在这里：写日志文件、发送告警、dump 堆栈等
    std::exit(static_cast<int>(code));
}

// 使用
void* allocate(size_t size) {
    void* ptr = malloc(size);
    if (!ptr) {
        fatal_error(ErrorCode::OutOfMemory, 
                    "Failed to allocate " + std::to_string(size) + " bytes");
        // ✅ 无警告：编译器知道上面不返回
    }else{
    	return ptr;
    }

}
```

# 5.注意事项
## （1）函数必须真的不返回
```cpp
// ❌ 未定义行为（UB）：函数可能会返回
[[noreturn]] void bad_exit(bool flag) {
    if (flag) {
        throw std::runtime_error("error");
    }
    // flag 为 false 时，函数会"返回" → 未定义行为！
    // 程序可能崩溃、产生随机结果
}

// ✅ 正确：确保所有路径都不返回
[[noreturn]] void good_exit(bool flag) {
    if (flag) {
        throw std::runtime_error("error");
    }
    std::abort();  // 兜底，保证不返回
}
```

## （2）所有分支都必须不返回
```cpp
// ❌ 危险：default 分支没有处理
[[noreturn]] void handle_error(int code) {
    switch (code) {
        case 1: throw std::runtime_error("error 1");
        case 2: throw std::logic_error("error 2");
        // default 分支缺失 → 未定义行为！
    }
}

// ✅ 安全：所有分支都有处理
[[noreturn]] void handle_error(int code) {
    switch (code) {
        case 1: throw std::runtime_error("error 1");
        case 2: throw std::logic_error("error 2");
        default: throw std::runtime_error("unknown error");
    }
}
```

## （3）声明和定义都要加
```cpp
// header.h
[[noreturn]] void fatal_error(const char* msg);  // ✅ 声明要加

// source.cpp
[[noreturn]] void fatal_error(const char* msg) {  // ✅ 定义也要加
    std::cerr << msg << std::endl;
    std::abort();
}
```