---
title: C++模板基础语法
date: 2026-01-19 14:47:02
tags:
	- 笔记
categories:
	- C++
	- C++特性
	- C++模板基础语法
---

# 一、函数模板
## 1.定义格式
```cpp
#include <iostream>
#include <string>

// 1.基本格式（类型参数）
template <typename T>
T add(T a, T b) {
    return a + b;
}

// 2.多类型参数
template <typename T1, typename T2>
auto max_val(T1 a, T2 b) {
    return a > b ? a : b;  // C++11后auto可推导返回值类型
}

// 3.非类型参数（必须是编译期可确定的常量，如整数、指针、引用）
template <typename T, int N>
T sum_array(T arr[N]) {
    T sum = 0;
    for (int i = 0; i < N; ++i) {
        sum += arr[i];
    }
    return sum;
}

// 4.默认模板参数（类型参数与非类型参数均可设置默认值）
// 规则：无默认值的参数在前，有默认值的参数在后，避免歧义

template <typename T, typename E = std::string>  // 类型参数默认值
void printPair(T a, E b) {
    std::cout << "Pair: " << a << ", " << b << std::endl;
}


template <typename T, int N = 5>  // 非类型参数默认值
T getMax(T arr[]) {
    T max_val = arr[0];
    for (int i = 1; i < N; ++i) {
        if (arr[i] > max_val) max_val = arr[i];
    }
    return max_val;
}

int main() {
    // === 1. 基本格式 ===
    std::cout << "=== 基本格式 ===" << std::endl;
    std::cout << add(3, 5) << std::endl;           // 自动推导：int
    std::cout << add(1.5, 2.3) << std::endl;       // 自动推导：double
    std::cout << add<int>(3, 5) << std::endl;      // 显式指定类型
    
    // === 2. 多类型参数 ===
    std::cout << "\n=== 多类型参数 ===" << std::endl;
    std::cout << max_val(3, 5.5) << std::endl;     // int vs double → double
    std::cout << max_val(10.2, 5) << std::endl;    // double vs int → double
    std::cout << max_val('A', 100) << std::endl;   // char vs int → int
    
    // === 3. 非类型参数 ===
    std::cout << "\n=== 非类型参数 ===" << std::endl;
    int arr1[] = {1, 2, 3, 4, 5};
    double arr2[] = {1.1, 2.2, 3.3};
    std::cout << sum_array(arr1) << std::endl;     // 自动推导 N=5，输出 15
    std::cout << sum_array(arr2) << std::endl;     // 自动推导 N=3，输出 6.6
    
    // === 4. 默认模板参数 ===
    std::cout << "\n=== 默认模板参数 ===" << std::endl;
    
    // printPair
    printPair(42, "hello");                        // T=int, E=string（默认）
    printPair(3.14, "world");                      // T=double, E=string（默认）
    printPair<int, int>(10, 20);                   // 显式指定 E=int
    
    // getMax
    int arr3[5] = {3, 7, 2, 9, 1};
    int arr4[3] = {100, 50, 75};
    std::cout << getMax(arr3) << std::endl;        // N=5（默认），输出 9
    std::cout << getMax<int, 3>(arr4) << std::endl;// 显式指定 N=3，输出 100
    
    return 0;
}
```

# 二、类模板
## 1.定义格式
```cpp
#include <iostream>
#include <string>
#include <vector>

// ===========================================
// 1. 基本格式（类型参数）
// ===========================================
template <typename T>
class Box {
private:
    T value;
public:
    Box(T v) : value(v) {}
    T get() const { return value; }
    void set(T v) { value = v; }
};

// ===========================================
// 2. 多类型参数
// ===========================================
template <typename K, typename V>
class Pair {
private:
    K key;
    V value;
public:
    Pair(K k, V v) : key(k), value(v) {}
    K getKey() const { return key; }
    V getValue() const { return value; }
};

// ===========================================
// 3. 非类型参数（编译期常量）
// ===========================================
template <typename T, int Size>
class Array {
private:
    T data[Size];
public:
    T& operator[](int index) { return data[index]; }
    int size() const { return Size; }
};

// ===========================================
// 4. 默认模板参数
// ===========================================

// 4.1 类型参数默认值
template <typename T, typename E = std::string>
class DataHolder {
private:
    T id;
    E description;
public:
    DataHolder(T i, E desc) : id(i), description(desc) {}
    void print() const {
        std::cout << "ID: " << id << ", Desc: " << description << std::endl;
    }
};

// 4.2 非类型参数默认值
template <typename T = int, int Capacity = 10>
class Stack {
private:
    T data[Capacity];
    int top = -1;
public:
    void push(T val) {
        if (top < Capacity - 1) data[++top] = val;
    }
    T pop() {
        return (top >= 0) ? data[top--] : T{};
    }
    bool empty() const { return top == -1; }
};

// 4.3 混合默认参数（类型 + 非类型）
template <typename T = int, typename E = std::string, int Capacity = 10>
class Container {
private:
    T keys[Capacity];
    E values[Capacity];
    int count = 0;
public:
    void add(T key, E value) {
        if (count < Capacity) {
            keys[count] = key;
            values[count] = value;
            count++;
        }
    }
    void printAll() const {
        for (int i = 0; i < count; ++i) {
            std::cout << keys[i] << " -> " << values[i] << std::endl;
        }
    }
    int size() const { return count; }
    int capacity() const { return Capacity; }
};

// ===========================================
// 5. 模板模板参数
// ===========================================
template <typename T, template<typename, typename> class Container>
class Wrapper {
private:
    Container<T, std::allocator<T>> container;
public:
    void add(const T& item) { container.push_back(item); }
    size_t size() const { return container.size(); }
};

// ===========================================
// 6. 类外定义成员函数
// ===========================================
template <typename T>
class Calculator {
private:
    T val;
public:
    Calculator(T v) : val(v) {}
    T add(T x);
    T multiply(T x);
};

template <typename T>
T Calculator<T>::add(T x) {
    return val + x;
}

template <typename T>
T Calculator<T>::multiply(T x) {
    return val * x;
}

// ===========================================
// 使用示例
// ===========================================
int main() {
    std::cout << "========== 1. 基本格式 ==========" << std::endl;
    Box<int> intBox(42);
    Box<std::string> strBox("Hello");
    Box<double> dblBox(3.14);
    
    std::cout << "intBox: " << intBox.get() << std::endl;
    std::cout << "strBox: " << strBox.get() << std::endl;
    std::cout << "dblBox: " << dblBox.get() << std::endl;
    
    intBox.set(100);
    std::cout << "intBox after set: " << intBox.get() << std::endl;

    std::cout << "\n========== 2. 多类型参数 ==========" << std::endl;
    Pair<std::string, int> p1("age", 25);
    Pair<int, double> p2(1, 99.99);
    Pair<std::string, std::string> p3("name", "Alice");
    
    std::cout << p1.getKey() << " = " << p1.getValue() << std::endl;
    std::cout << p2.getKey() << " = " << p2.getValue() << std::endl;
    std::cout << p3.getKey() << " = " << p3.getValue() << std::endl;

    std::cout << "\n========== 3. 非类型参数 ==========" << std::endl;
    Array<int, 5> arr1;
    Array<double, 3> arr2;
    
    for (int i = 0; i < arr1.size(); ++i) {
        arr1[i] = i * 10;
    }
    arr2[0] = 1.1; arr2[1] = 2.2; arr2[2] = 3.3;
    
    std::cout << "arr1 (size=" << arr1.size() << "): ";
    for (int i = 0; i < arr1.size(); ++i) {
        std::cout << arr1[i] << " ";
    }
    std::cout << std::endl;
    
    std::cout << "arr2 (size=" << arr2.size() << "): ";
    for (int i = 0; i < arr2.size(); ++i) {
        std::cout << arr2[i] << " ";
    }
    std::cout << std::endl;

    std::cout << "\n========== 4.1 类型参数默认值 ==========" << std::endl;
    DataHolder<int> d1(1, "First Item");           // E = std::string（默认）
    DataHolder<double> d2(3.14, "Pi Value");       // E = std::string（默认）
    DataHolder<int, int> d3(100, 200);             // E = int（显式指定）
    DataHolder<std::string, double> d4("price", 99.99);
    
    d1.print();
    d2.print();
    d3.print();
    d4.print();

    std::cout << "\n========== 4.2 非类型参数默认值 ==========" << std::endl;
    Stack<> s1;                  // T=int, Capacity=10（全部默认）
    Stack<double> s2;            // T=double, Capacity=10（默认）
    Stack<char, 5> s3;           // T=char, Capacity=5
    
    s1.push(10); s1.push(20); s1.push(30);
    std::cout << "s1 pop: " << s1.pop() << ", " << s1.pop() << std::endl;
    
    s2.push(1.1); s2.push(2.2);
    std::cout << "s2 pop: " << s2.pop() << std::endl;
    
    s3.push('A'); s3.push('B'); s3.push('C');
    std::cout << "s3 pop: " << s3.pop() << ", " << s3.pop() << std::endl;

    std::cout << "\n========== 4.3 混合默认参数 ==========" << std::endl;
    Container<> c1;                              // T=int, E=string, Capacity=10
    Container<double> c2;                        // T=double, E=string, Capacity=10
    Container<int, int> c3;                      // T=int, E=int, Capacity=10
    Container<std::string, double, 5> c4;        // 全部显式指定
    
    c1.add(1, "apple");
    c1.add(2, "banana");
    std::cout << "c1 (capacity=" << c1.capacity() << "):" << std::endl;
    c1.printAll();
    
    c2.add(1.1, "one point one");
    c2.add(2.2, "two point two");
    std::cout << "c2:" << std::endl;
    c2.printAll();
    
    c3.add(1, 100);
    c3.add(2, 200);
    std::cout << "c3:" << std::endl;
    c3.printAll();
    
    c4.add("pi", 3.14159);
    c4.add("e", 2.71828);
    std::cout << "c4 (capacity=" << c4.capacity() << "):" << std::endl;
    c4.printAll();

    std::cout << "\n========== 5. 模板模板参数 ==========" << std::endl;
    Wrapper<int, std::vector> w1;
    Wrapper<std::string, std::vector> w2;
    
    w1.add(10); w1.add(20); w1.add(30);
    w2.add("hello"); w2.add("world");
    
    std::cout << "w1 size: " << w1.size() << std::endl;
    std::cout << "w2 size: " << w2.size() << std::endl;

    std::cout << "\n========== 6. 类外定义成员函数 ==========" << std::endl;
    Calculator<int> calc1(10);
    Calculator<double> calc2(3.5);
    
    std::cout << "calc1: 10 + 5 = " << calc1.add(5) << std::endl;
    std::cout << "calc1: 10 * 3 = " << calc1.multiply(3) << std::endl;
    std::cout << "calc2: 3.5 + 1.5 = " << calc2.add(1.5) << std::endl;
    std::cout << "calc2: 3.5 * 2 = " << calc2.multiply(2) << std::endl;

    return 0;
}
```

## 2.模板类的继承
模板类可作为基类，子类可分为“普通子类”和“模板子类”，需注意模板参数的传递。
```cpp
// 模板类作为基类
template <typename T>
class Base {
protected:
    T value;
public:
    Base(T val) : value(val) {}
};

// 1.普通子类（需指定基类的模板参数）
class Derived1 : public Base<int> {
public:
    Derived1(int val) : Base<int>(val) {}
};

// 2.模板子类（可复用父类的模板参数，也可新增参数）
template <typename T, typename U>
class Derived2 : public Base<T> {
private:
    U extra;
public:
    Derived2(T val1, U val2) : Base<T>(val1), extra(val2) {}
};
```

# 三、模板特化
模板特化是针对特定模板参数（类型/值），提供定制化的实现逻辑，解决通用模板在特定场景下的适配问题。特化分为全特化和偏特化。

## 1.全特化
- 全特化是对模板的**“所有参数”都指定“具体值/类型”**，本质是模板的一个“特例实现”。
```cpp
// 通用模板
template <typename T1, typename T2>
class Pair {
private:
    T1 first;
    T2 second;
public:
    Pair(T1 f, T2 s) : first(f), second(s) {}
    void print() {
        std::cout << "Generic Pair: " << first << ", " << second << std::endl;
    }
};

// 全特化（T1=int，T2=double）
template <>  // 全特化无模板参数
class Pair<int, double> {
private:
    int first;
    double second;
public:
    Pair(int f, double s) : first(f), second(s) {}
    void print() {
        std::cout << "Specialized Pair (int, double): " << first << ", " << second << std::endl;
    }
};

int main() {
    Pair<std::string, int> p1("hello", 1);
    p1.print();  // 调用通用模板

    Pair<int, double> p2(10, 3.14);
    p2.print();  // 调用全特化版本
    return 0;
}
```

## 2.偏特化
- 偏特化是对模板的**“部分参数”指定“具体值/类型”**，或对参数进行“限定”（如指针、引用、const修饰），仍保留部分泛型参数。
	- 仅类模板支持偏特化，函数模板不支持偏特化（可通过重载替代）。
```cpp
// 通用模板
template <typename T1, typename T2>
class Pair { ... };

// 偏特化1：部分参数指定（T2=std::string）
template <typename T1>
class Pair<T1, std::string> {
public:
    void print() {
        std::cout << "Partial Specialization (T1, string)" << std::endl;
    }
};

// 偏特化2：参数限定为指针类型
template <typename T1, typename T2>
class Pair<T1*, T2*> {  // 两个参数均为指针
public:
    void print() {
        std::cout << "Partial Specialization (T1*, T2*)" << std::endl;
    }
};

// 偏特化3：参数限定为const类型
template <typename T1, typename T2>
class Pair<const T1, T2> {
public:
    void print() {
        std::cout << "Partial Specialization (const T1, T2)" << std::endl;
    }
};
```

# 四、实战补充示例
## 1. <> 的含义：使用所有模板参数的默认值
```cpp
template<typename T = int, typename E = std::string>
using Result = std::pair<T, E>;

Result<>           // = Result<int, std::string>
Result<double>     // = Result<double, std::string>
Result<double, X>  // = Result<double, X>
```