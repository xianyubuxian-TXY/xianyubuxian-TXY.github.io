---
title: C++四大类型转换
date: 2026-01-26 20:58:43
tags:
	- 笔记
categories:
	- C++
	- 开发笔记
	- C++四大类型转换
---

# 0.总览对比
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126220659275.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126221959123.png)

**🎯 选择决策树**
```cpp
需要类型转换？
    │
    ├─► 只是添加/移除 const？
    │       └─► const_cast
    │
    ├─► 继承体系中的向下转换，需要运行时安全检查？
    │       └─► dynamic_cast
    │
    ├─► 相关类型间转换（数值、枚举、继承、void*）？
    │       └─► static_cast ⭐ 首选
    │
    └─► 完全不相关的类型，底层位操作？
            └─► reinterpret_cast（三思！）
```

## 为什么不直接用 C 风格转换？
- C 风格转换会**按顺序尝试** const_cast → static_cast → reinterpret_cast，容易**隐藏错误**
```cpp
int x = 10;
const int* cp = &x;

// C 风格转换：一行能做多件事，但你不知道它做了什么
char* p = (char*)cp;  // 同时移除 const + 重新解释类型！

// C++ 风格：意图明确，更容易发现错误
// char* p = const_cast<char*>(cp);          // ❌ 编译失败，类型不同
// char* p = reinterpret_cast<char*>(cp);    // ❌ 编译失败，还有 const
char* p = reinterpret_cast<char*>(const_cast<int*>(cp));  // ✅ 两步显式操作
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126221416116.png)

# 1.static_cast
## （1）主要用途
- 编译期的**相关类型**间显式转换，是最常用的转换。

## （2）失败时的表现
- **编译期**：
	- **不相关类型**会编译失败
- **运行期**：
	- 不检查，错误使用导致未定义行为（**不报错，直接崩或产生垃圾值**）

### <1>编译期失败：不相关类型无法转换
```cpp
#include <string>

int main() {
    // ❌ 编译失败：int* 和 double* 是不相关的指针类型
    int* pi = nullptr;
    double* pd = static_cast<double*>(pi);  // error: invalid static_cast

    // ❌ 编译失败：std::string 和 int 没有转换关系
    std::string s = "hello";
    int n = static_cast<int>(s);  // error: invalid static_cast

    // ❌ 编译失败：不相关的类之间
    class A {};
    class B {};  // A 和 B 没有继承关系
    
    A a;
    B* pb = static_cast<B*>(&a);  // error: invalid static_cast
}

```

### <2> 运行期失败：未定义行为（编译通过，但结果错误）
#### 1）向下转型（父类指针 → 子类指针）转错了
```cpp
#include <iostream>

class Base {
public:
    virtual ~Base() = default;
    int base_val = 10;
};

class DerivedA : public Base {
public:
    int a_val = 100;
    void funcA() { std::cout << "DerivedA: " << a_val << std::endl; }
};

class DerivedB : public Base {
public:
    double b_val = 3.14;
    void funcB() { std::cout << "DerivedB: " << b_val << std::endl; }
};

int main() {
    Base* ptr = new DerivedB();  // 实际指向 DerivedB 对象
    
    // ⚠️ 编译通过！但这是错误的转换（实际是 DerivedB，却转成 DerivedA*）
    DerivedA* pa = static_cast<DerivedA*>(ptr);
    
    // 💥 未定义行为：访问了不存在的成员
    pa->funcA();   // 可能崩溃、输出垃圾值、或者"看起来正常"
    std::cout << pa->a_val << std::endl;  // 垃圾值
    
    delete ptr;
}
```
**可能的输出**（取决于内存布局）：
```bash
DerivedA: 1074339512    ← 垃圾值（把 double 的内存当 int 读了）
1074339512
```

#### 2）数值截断（大范围 → 小范围）
```cpp
#include <iostream>

int main() {
    long long big = 5'000'000'000LL;  // 50亿，超出 int 范围
    
    // ⚠️ 编译通过！但值会被截断
    int small = static_cast<int>(big);
    
    std::cout << "原值: " << big << std::endl;
    std::cout << "转换后: " << small << std::endl;  // 不是50亿！
}
```
**输出**：
```bash
原值: 5000000000
转换后: 705032704    ← 被截断了，静默丢失数据
```

#### 3）引用转换错误
```cpp
#include <iostream>

class Animal { public: virtual ~Animal() = default; };
class Dog : public Animal { public: void bark() { std::cout << "Woof!\n"; } };
class Cat : public Animal { public: void meow() { std::cout << "Meow!\n"; } };

int main() {
    Cat cat;
    Animal& animal_ref = cat;  // 实际是 Cat
    
    // ⚠️ 编译通过！但转换是错的
    Dog& dog_ref = static_cast<Dog&>(animal_ref);
    
    // 💥 未定义行为
    dog_ref.bark();  // 可能崩溃或执行错误代码
}
```

## （3）使用场景
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126212944696.png)

### <1>使用示例
```cpp
// 数值转换
double pi = 3.14159;
int i = static_cast<int>(pi);  // 3

// 枚举转换
enum class Color { Red = 1, Green, Blue };
int val = static_cast<int>(Color::Green);  // 2
Color c = static_cast<Color>(3);           // Blue

// void* 还原
int x = 42;
void* vp = &x;
int* ip = static_cast<int*>(vp);  // 还原回 int*

// 继承体系转换
class Base { public: virtual ~Base() = default; };
class Derived : public Base { public: int data = 100; };

Derived d;
Base* bp = static_cast<Base*>(&d);      // ✅ 向上转换，安全

Base* bp2 = new Derived();
Derived* dp = static_cast<Derived*>(bp2);  // ⚠️ 向下转换，不检查
```

### <2>⚠️ 注意事项
```cpp
// 💥 危险：实际类型不匹配
Base* bp = new Base();  // 实际是 Base，不是 Derived
Derived* dp = static_cast<Derived*>(bp);  // 编译通过！
dp->data = 10;  // 💥 未定义行为！访问了不存在的成员

// ✅ 原则：向下转换前必须确定实际类型
```

# 2.dynamic_cast
## （1）主要用途
- **运行时**安全检查的**多态类型转换**，专门用于**“继承体系”中的向下（父类——>子类）/ 交叉转换**。
- static_cast 也可以用于多态类型（有继承关系的类）的转换，但**安全性由程序员自己负责**。
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126213238272.png)

## （2）失败时的表现
- **指针转换**：
	- 返回 `nullptr`
- **引用转换**：
	- 抛出 `std::bad_cast` 异常

### <1>指针转换失败：返回 nullptr
```cpp
#include <iostream>

class Base {
public:
    virtual ~Base() = default;  // 必须有虚函数才能用 dynamic_cast
};

class DerivedA : public Base {
public:
    void funcA() { std::cout << "I am DerivedA\n"; }
};

class DerivedB : public Base {
public:
    void funcB() { std::cout << "I am DerivedB\n"; }
};

int main() {
    Base* ptr = new DerivedB();  // 实际指向 DerivedB
    
    // 尝试转换为 DerivedA*（错误的类型）
    DerivedA* pa = dynamic_cast<DerivedA*>(ptr);
    
    if (pa == nullptr) {
        std::cout << "转换失败！pa 是 nullptr\n";
    } else {
        pa->funcA();
    }
    
    // 尝试转换为 DerivedB*（正确的类型）
    DerivedB* pb = dynamic_cast<DerivedB*>(ptr);
    
    if (pb != nullptr) {
        std::cout << "转换成功！";
        pb->funcB();
    }
    
    delete ptr;
}
```
**输出**：
```bash
转换失败！pa 是 nullptr
转换成功！I am DerivedB
```

### <2>引用转换失败：抛出 std::bad_cast
```cpp
#include <iostream>
#include <typeinfo>  // std::bad_cast

class Base {
public:
    virtual ~Base() = default;
};

class DerivedA : public Base {};
class DerivedB : public Base {};

int main() {
    DerivedB b;
    Base& base_ref = b;  // 实际是 DerivedB
    
    try {
        // 尝试转换为 DerivedA&（错误的类型）
        DerivedA& a_ref = dynamic_cast<DerivedA&>(base_ref);
        
        // 不会执行到这里
        std::cout << "转换成功\n";
        
    } catch (const std::bad_cast& e) {
        std::cout << "转换失败！捕获异常: " << e.what() << std::endl;
    }
}
```
**输出**：
```bash
转换失败！捕获异常: std::bad_cast
```

## （3）使用规范
### <1>指针转换：用 if 检查
```cpp
// ✅ 推荐写法
void process(Base* ptr) {
    if (Derived* d = dynamic_cast<Derived*>(ptr)) {
        // 转换成功，d 可用
        d->derivedMethod();
    } else {
        // 转换失败，处理其他情况
    }
}

// ❌ 不要这样写（不检查就用）
void bad_process(Base* ptr) {
    Derived* d = dynamic_cast<Derived*>(ptr);
    d->derivedMethod();  // 如果 d 是 nullptr，崩溃！
}
```

### <2>引用转换：用 try-catch
```cpp
// ✅ 推荐写法
void process(Base& ref) {
    try {
        Derived& d = dynamic_cast<Derived&>(ref);
        d.derivedMethod();
    } catch (const std::bad_cast&) {
        // 转换失败
    }
}

// 或者：先用指针判断，再用引用
void process_v2(Base& ref) {
    if (dynamic_cast<Derived*>(&ref)) {
        Derived& d = dynamic_cast<Derived&>(ref);  // 这次一定成功
        d.derivedMethod();
    }
}
```

## （4）典型使用模式
### <1>类型安全的向下转型
```cpp
class Animal {
public:
    virtual ~Animal() = default;  // 必须有虚函数！
    virtual void speak() = 0;
};

class Dog : public Animal {
public:
    void speak() override { std::cout << "Woof!\n"; }
    void fetch() { std::cout << "Fetching ball!\n"; }  // Dog 特有
};

class Cat : public Animal {
public:
    void speak() override { std::cout << "Meow!\n"; }
    void climb() { std::cout << "Climbing tree!\n"; }  // Cat 特有
};

// 指针转换示例
void try_fetch(Animal* animal) {
    if (Dog* dog = dynamic_cast<Dog*>(animal)) {
        dog->fetch();  // ✅ 安全：确定是 Dog
    } else {
        std::cout << "Not a dog, can't fetch\n";
    }
}

// 引用转换示例
void must_be_cat(Animal& animal) {
    try {
        Cat& cat = dynamic_cast<Cat&>(animal);
        cat.climb();
    } catch (const std::bad_cast& e) {
        std::cout << "Not a cat: " << e.what() << "\n";
    }
}

// 使用
Animal* a1 = new Dog();
Animal* a2 = new Cat();

try_fetch(a1);  // 输出：Fetching ball!
try_fetch(a2);  // 输出：Not a dog, can't fetch
```

### <2>交叉转换

```cpp
class Printable { 
public: 
    virtual ~Printable() = default;
    virtual void print() = 0; 
};

class Serializable { 
public: 
    virtual ~Serializable() = default;
    virtual void serialize() = 0; 
};

class Document : public Printable, public Serializable {
public:
    void print() override { std::cout << "Printing\n"; }
    void serialize() override { std::cout << "Serializing\n"; }
};

void cross_cast_demo() {
    Document doc;
    Printable* pp = &doc;
    
    // Printable* → Serializable*（兄弟类，非继承关系）
    Serializable* sp = dynamic_cast<Serializable*>(pp);
    if (sp) {
        sp->serialize();  // ✅ 成功！运行时找到了实际类型 Document
    }
}
```

## （5）⚠️ 注意事项
- 没有虚函数无法使用
- 性能开销：
	- 有运行时检查，**比 static_cast 慢**
- 优先用多态而非 dynamic_cast

# 3.const_cast
## （1）主要用途
**唯一**能**“添加”**或**“移除”** const / volatile 限定符的转换。

## （2）支持类型
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126220227747.png)
```cpp
#include <iostream>

int main() {
    const int x = 100;
    
    // ✅ 1. 指针
    const int* cp = &x;
    int* p = const_cast<int*>(cp);
    
    // ✅ 2. 引用
    const int& cr = x;
    int& r = const_cast<int&>(cr);
    
    // ❌ 3. 值类型 —— 没意义，编译失败
    // int y = const_cast<int>(x);  // error!
    
    std::cout << "通过指针: " << *p << "\n";
    std::cout << "通过引用: " << r << "\n";
}
```

### 为什么“值类型”不行？
- 值拷贝会创建新对象，新对象的 const 属性由你自己声明决定，和原对象无关
```cpp
const int x = 10;
int y = x;  // 这本身就是拷贝，y 已经是非 const 了，不需要 cast
```


## （3）失败时的表现
- **编译期**：
	- 非 const 相关的转换会失败
- **运行期**：
	- 若修改了真正的 const 对象，产生未定义行为

### <1>编译期失败：非 const 相关的转换
```cpp
#include <iostream>

class Base {};
class Derived : public Base {};

int main() {
    // ❌ 错误1：尝试转换不相关的类型
    int x = 10;
    // double* p = const_cast<double*>(&x);  // 编译错误！
    // error: invalid const_cast from type 'int*' to type 'double*'
    
    // ❌ 错误2：尝试做继承转换
    Derived d;
    // Base* b = const_cast<Base*>(&d);  // 编译错误！
    // error: invalid const_cast（这应该用 static_cast）
    
    // ❌ 错误3：尝试转换指针为整数
    // int addr = const_cast<int>(&x);  // 编译错误！
    
    // ✅ 正确：只处理 const
    const int y = 20;
    int* py = const_cast<int*>(&y);  // OK，移除 const
    
    const int* cp = &x;
    int* p = const_cast<int*>(cp);   // OK，移除 const
    
    int* np = &x;
    const int* cnp = const_cast<const int*>(np);  // OK，添加 const
}
```
**编译错误信息示例**：
```bash
error: invalid const_cast from type 'int*' to type 'double*'
```


### <2>运行期：修改真正的 const 对象 → 未定义行为
```cpp
#include <iostream>

int main() {
    // ============ 情况1：安全 ============
    // 原对象本身不是 const，只是通过 const 指针访问
    int normal_var = 100;
    const int* cp1 = &normal_var;  // 用 const 指针指向非 const 对象
    
    int* p1 = const_cast<int*>(cp1);
    *p1 = 200;  // ✅ 安全！原对象本身是可修改的
    
    std::cout << "normal_var = " << normal_var << "\n";  // 输出 200
    
    
    // ============ 情况2：未定义行为 ============
    // 原对象本身就是 const
    const int const_var = 100;  // 真正的 const 对象
    
    int* p2 = const_cast<int*>(&const_var);
    *p2 = 200;  // ⚠️ 未定义行为！修改了真正的 const 对象
    
    // 可能的奇怪结果：
    std::cout << "const_var = " << const_var << "\n";  // 可能输出 100！
    std::cout << "*p2 = " << *p2 << "\n";              // 可能输出 200！
    // 同一个地址，不同的值？！—— 这就是 UB
}
```
**可能的输出（取决于编译器优化）**：
```bash
normal_var = 200
const_var = 100    ← 编译器可能直接用常量替换
*p2 = 200          ← 但内存确实被改了
```

## （4）使用场景
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126215229755.png)

### <1> 调用旧 API
- 老接口参数不是 const，但你知道它内部**不会修改数据**。
```cpp
#include <iostream>
#include <cstring>

// 假设这是一个旧的 C 库函数（你无法修改它）
void legacy_print(char* str) {  // 参数不是 const，但实际只读取
    printf("Legacy output: %s\n", str);
}

// 旧的长度计算函数
size_t legacy_strlen(char* str) {  // 同样不是 const
    size_t len = 0;
    while (*str++) len++;
    return len;
}

// 现代 C++ 代码
void modern_function(const char* message) {
    // ❌ 直接调用会编译错误
    // legacy_print(message);  // error: invalid conversion from 'const char*' to 'char*'
    
    // ✅ 用 const_cast（你确认 legacy_print 不会修改）
    legacy_print(const_cast<char*>(message));
    
    size_t len = legacy_strlen(const_cast<char*>(message));
    std::cout << "Length: " << len << "\n";
}

int main() {
    const char* msg = "Hello, World!";
    modern_function(msg);
}
```

### <2>const 重载复用
- 避免在 const 和非 const 版本中写重复代码。
```cpp
#include <iostream>
#include <vector>

class Matrix {
    std::vector<std::vector<double>> data_;
    int rows_, cols_;
    
public:
    Matrix(int r, int c) : rows_(r), cols_(c), data_(r, std::vector<double>(c, 0)) {}
    
    // 非 const 版本：包含完整逻辑
    double& at(int row, int col) {
        // 边界检查（假设这里有很多复杂逻辑）
        if (row < 0 || row >= rows_ || col < 0 || col >= cols_) {
            throw std::out_of_range("Index out of bounds");
        }
        std::cout << "[非const版本被调用]\n";
        return data_[row][col];
    }
    
    // const 版本：复用非 const 版本，避免代码重复
    const double& at(int row, int col) const {
        std::cout << "[const版本被调用] -> ";
        // 1. const_cast 移除 this 的 const
        // 2. 调用非 const 版本
        // 3. 返回值自动转为 const 引用
        return const_cast<Matrix*>(this)->at(row, col);
    }
};

int main() {
    Matrix m(3, 3);
    m.at(1, 1) = 5.0;         // 调用非 const 版本
    
    const Matrix& cm = m;
    double val = cm.at(1, 1); // 调用 const 版本
    std::cout << "Value: " << val << "\n";
}
```
**输出**：
```bash
[非const版本被调用]
[const版本被调用] -> [非const版本被调用]
Value: 5
```

### <3>临时移除 const（缓存/懒加载）
- 使用 mutable 更好，但有时需要 const_cast：
```cpp
#include <iostream>
#include <string>
#include <optional>

class DataProcessor {
    std::string raw_data_;
    
    // mutable 允许在 const 成员函数中修改
    mutable std::optional<int> cached_result_;
    mutable bool cache_valid_ = false;
    
public:
    DataProcessor(const std::string& data) : raw_data_(data) {}
    
    // 逻辑上是 const（不改变对象的"可观察状态"）
    // 但内部会更新缓存
    int getProcessedValue() const {
        if (!cache_valid_) {
            std::cout << "[执行耗时计算...]\n";
            // 模拟复杂计算
            int result = 0;
            for (char c : raw_data_) result += c;
            
            cached_result_ = result;  // mutable 成员可以修改
            cache_valid_ = true;
        }
        return *cached_result_;
    }
};

// 如果没有 mutable，需要用 const_cast（不推荐）
class DataProcessor_NoMutable {
    std::string raw_data_;
    std::optional<int> cached_result_;  // 没有 mutable
    bool cache_valid_ = false;
    
public:
    DataProcessor_NoMutable(const std::string& data) : raw_data_(data) {}
    
    int getProcessedValue() const {
        if (!cache_valid_) {
            std::cout << "[执行耗时计算（const_cast 版本）...]\n";
            
            int result = 0;
            for (char c : raw_data_) result += c;
            
            // ⚠️ 用 const_cast 移除 this 的 const
            auto* self = const_cast<DataProcessor_NoMutable*>(this);
            self->cached_result_ = result;
            self->cache_valid_ = true;
        }
        return *cached_result_;
    }
};

int main() {
    std::cout << "=== mutable 版本 ===\n";
    const DataProcessor dp("Hello");
    std::cout << dp.getProcessedValue() << "\n";  // 计算
    std::cout << dp.getProcessedValue() << "\n";  // 使用缓存
    
    std::cout << "\n=== const_cast 版本 ===\n";
    const DataProcessor_NoMutable dp2("Hello");
    std::cout << dp2.getProcessedValue() << "\n";
    std::cout << dp2.getProcessedValue() << "\n";
}
```
**输出**：
```bash
=== mutable 版本 ===
[执行耗时计算...]
500
500

=== const_cast 版本 ===
[执行耗时计算（const_cast 版本）...]
500
500
```

### <4>补充：移除 / 添加 volatile
```cpp
// const_cast 也可以处理 volatile
int x = 10;
volatile int* vp = &x;

// 移除 volatile
int* p = const_cast<int*>(vp);

// 添加 volatile
int y = 20;
volatile int* vp2 = const_cast<volatile int*>(&y);
```

# 4.reinterpret_cast
## （1）主要用途
- 最底层、最危险的转换，将**位模式**重新解释为另一种完全不相关的类型。

## （2）失败时的表现
- **编译期**：
	- 几乎不检查，大部分都能编译
- **运行期**：
	- 错误使用直接导致**未定义行为**、**崩溃**、**数据损坏**

## （3）使用场景
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260126220435635.png)
```cpp
// 场景 1：指针与整数互转
int* ptr = new int(42);
uintptr_t addr = reinterpret_cast<uintptr_t>(ptr);  // 保存地址
std::cout << "Address: 0x" << std::hex << addr << "\n";
int* restored = reinterpret_cast<int*>(addr);       // 恢复指针
std::cout << "Value: " << *restored << "\n";        // 42

// 场景 2：查看对象的原始字节
struct Point { float x, y; };
Point p{1.0f, 2.0f};

auto* bytes = reinterpret_cast<unsigned char*>(&p);
std::cout << "Raw bytes: ";
for (size_t i = 0; i < sizeof(p); ++i) {
    std::cout << std::hex << static_cast<int>(bytes[i]) << " ";
}

// 场景 3：硬件/嵌入式编程（内存映射 IO）
// volatile uint32_t* gpio = reinterpret_cast<volatile uint32_t*>(0x40020000);
// *gpio = 0x01;  // 写硬件寄存器

// 场景 4：处理特定格式的数据包
struct NetworkPacket {
    uint8_t header[4];
    uint32_t payload_size;
    // ...
};

void process_buffer(char* buffer, size_t len) {
    auto* packet = reinterpret_cast<NetworkPacket*>(buffer);
    // ⚠️ 需要注意对齐问题！
}
```

## （4）⚠️ 注意事项
```cpp
// 💥 违反严格别名规则 (Strict Aliasing)
float f = 3.14f;
int* ip = reinterpret_cast<int*>(&f);
int i = *ip;  // 💥 未定义行为！

// ✅ 正确做法 1：使用 memcpy
int i2;
std::memcpy(&i2, &f, sizeof(f));

// ✅ 正确做法 2：C++20 的 std::bit_cast
#include <bit>
int i3 = std::bit_cast<int>(f);

// ⚠️ 对齐问题
char buffer[100];
// int* ip = reinterpret_cast<int*>(buffer + 1);  // ⚠️ 可能未对齐！
// *ip = 42;  // 某些平台会崩溃

// ⚠️ 不要用于类层次转换
class Base {};
class Derived : public Base { int x; };
Base b;
// Derived* dp = reinterpret_cast<Derived*>(&b);  // 💥 应该用 static/dynamic_cast
```