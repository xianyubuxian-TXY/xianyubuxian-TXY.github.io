---
title: json的使用
date: 2026-01-15 22:19:59
tags:
	- 笔记
categories:
	- C++
	- C++后端开发库
	- json
---

# 一、json库
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115222201657.png)

# 二、nlohmann/json
**注意**：
	- 所有操作需引入头文件 #include <nlohmann/json.hpp>，并使用命名空间 using json = nlohmann::json;

## 1.json支持的C++数据类型（直接转换）

### （1）基本数据类型 支持列表
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115223125969.png)
```cpp
#include <iostream>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

int main() {
    // ========== C++ → JSON ==========
    
    // 字符串
    json j_str = "hello world";
    json j_str2 = std::string("hello");
    
    // 整数
    json j_int = 42;
    json j_long = 9876543210LL;
    
    // 浮点数
    json j_float = 3.14;
    json j_double = 2.718281828;
    
    // 布尔
    json j_bool_t = true;
    json j_bool_f = false;
    
    // 空值
    json j_null = nullptr;
    
    // ========== JSON → C++ ==========
    
    std::string s = j_str.get<std::string>();       // "hello world"
    int i = j_int.get<int>();                       // 42
    double d = j_double.get<double>();              // 2.718281828
    bool b = j_bool_t.get<bool>();                  // true
    
    // 也可以用隐式转换
    std::string s2 = j_str;
    int i2 = j_int;
    
    // ========== 输出验证 ==========
    std::cout << "字符串: " << j_str << std::endl;
    std::cout << "整数:   " << j_int << std::endl;
    std::cout << "浮点数: " << j_double << std::endl;
    std::cout << "布尔:   " << j_bool_t << std::endl;
    std::cout << "空值:   " << j_null << std::endl;

   	return 0;
}
```

### （2）STL容器 支持列表
#### 1）序列容器 <——> JSON 数组
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115223530082.png)

```cpp
#include <iostream>
#include <nlohmann/json.hpp>
#include <vector>
#include <list>
#include <deque>
#include <array>
#include <forward_list>

using json = nlohmann::json;

int main() {
    // ========== C++ STL 容器 → JSON 数组 ==========
    
    // std::vector
    std::vector<int> vec = {1, 2, 3};
    json j_vec = vec;
    
    // std::list
    std::list<std::string> lst = {"apple", "banana", "cherry"};
    json j_lst = lst;
    
    // std::deque
    std::deque<double> deq = {1.1, 2.2, 3.3};
    json j_deq = deq;
    
    // std::array
    std::array<int, 4> arr = {10, 20, 30, 40};
    json j_arr = arr;
    
    // std::forward_list
    std::forward_list<bool> fwd = {true, false, true};
    json j_fwd = fwd;
    
    // ========== JSON 数组 → C++ STL 容器 ==========
    
    json j = {100, 200, 300, 400, 500};
    
    auto vec2 = j.get<std::vector<int>>();
    auto lst2 = j.get<std::list<int>>();
    auto deq2 = j.get<std::deque<int>>();
    auto fwd2 = j.get<std::forward_list<int>>();
    // std::array 需要大小匹配
    auto arr2 = j.get<std::array<int, 5>>();
    
    return 0;
}

```

#### 2）关联容器 <——> JSON 对象/数组
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115223609650.png)
```cpp
#include <iostream>
#include <nlohmann/json.hpp>
#include <map>
#include <unordered_map>
#include <set>
#include <unordered_set>

using json = nlohmann::json;

int main() {
    // ==================== C++ → JSON ====================
    
    // map<string, T> → JSON 对象 （Key 必须是 std::string）
    std::map<std::string, int> str_map = {{"apple", 10}, {"banana", 20}};
    json j_map = str_map;
    
    // unordered_map<string, T> → JSON 对象 （Key 必须是 std::string）
    std::unordered_map<std::string, double> str_umap = {{"pi", 3.14}, {"e", 2.71}};
    json j_umap = str_umap;
    
    // map<int, T> → JSON 数组 [[k,v], ...]
    std::map<int, std::string> int_map = {{1, "one"}, {2, "two"}};
    json j_int_map = int_map;
    
    // set<T> → JSON 数组
    std::set<int> s = {3, 1, 4, 1, 5};
    json j_set = s;
    
    // unordered_set<T> → JSON 数组
    std::unordered_set<std::string> us = {"red", "green", "blue"};
    json j_uset = us;
    
    // multimap / multiset → JSON 数组
    std::multiset<int> ms = {1, 1, 2, 2, 3};
    json j_mset = ms;
    
    // ==================== JSON → C++ ====================
    
    // JSON 对象 → map / unordered_map
    json obj = R"({"x": 100, "y": 200, "z": 300})"_json;
    auto map_back = obj.get<std::map<std::string, int>>();
    auto umap_back = obj.get<std::unordered_map<std::string, int>>();
    
    // JSON 数组 [[k,v],...] → map<int, T>
    json arr_kv = R"([[1, "a"], [2, "b"], [3, "c"]])"_json;
    auto int_map_back = arr_kv.get<std::map<int, std::string>>();
    
    // JSON 数组 → set / unordered_set
    json arr = R"([5, 3, 1, 4, 1, 5, 9])"_json;
    auto set_back = arr.get<std::set<int>>();           // 去重有序: {1,3,4,5,9}
    auto uset_back = arr.get<std::unordered_set<int>>(); // 去重无序
    
    // JSON 数组 → multiset (保留重复)
    auto mset_back = arr.get<std::multiset<int>>();     // {1,1,3,4,5,5,9}
    
    return 0;
}
```

#### 3）其他容器/类型
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115223649449.png)
```cpp
#include <iostream>
#include <nlohmann/json.hpp>
#include <utility>
#include <tuple>
#include <optional>
#include <string>

using json = nlohmann::json;

int main() {
    // ==================== C++ → JSON ====================
    
    // std::pair → JSON 数组 [key, value]
    std::pair<std::string, int> p = {"age", 25};
    json j_pair = p;
    
    std::pair<int, double> p2 = {42, 3.14};
    json j_pair2 = p2;
    
    // std::tuple → JSON 数组 [a, b, c, ...]
    std::tuple<int, std::string, bool> t = {100, "hello", true};
    json j_tuple = t;
    
    std::tuple<double, double, double> t2 = {1.1, 2.2, 3.3};
    json j_tuple2 = t2;
    
    // std::optional → T 或 null
    std::optional<int> opt_val = 42;
    std::optional<int> opt_empty = std::nullopt;
    std::optional<std::string> opt_str = "hello";
    
    json j_opt_val = opt_val;      // 42
    json j_opt_empty = opt_empty;  // null
    json j_opt_str = opt_str;      // "hello"
    
    // ==================== JSON → C++ ====================
    
    // JSON 数组 → std::pair
    json jp = R"(["name", 100])"_json;
    auto pair_back = jp.get<std::pair<std::string, int>>();
    
    // JSON 数组 → std::tuple
    json jt = R"([200, "world", false, 9.99])"_json;
    auto tuple_back = jt.get<std::tuple<int, std::string, bool, double>>();
    
    // JSON → std::optional
    json jv = 999;
    json jn = nullptr;
    
    auto opt_back = jv.get<std::optional<int>>();       // has_value() = true
    auto opt_null_back = jn.get<std::optional<int>>(); // has_value() = false
    
    return 0;
}
```

## 2.复杂类型与json的转换
**json与不少复杂类型都无法直接进行转换，这里主要列举了最常用到了几个复杂类型**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115230954500.png)
```cpp
#include <iostream>
#include <nlohmann/json.hpp>
#include <memory>
#include <optional>
#include <variant>
#include <chrono>
#include <stack>
#include <queue>

using json = nlohmann::json;

// ============================================================
//1.枚举类型
// ============================================================
//方法一：使用宏（推荐，最简洁）
// 定义枚举
enum class Color {
    Red,
    Green,
    Blue,
    Unknown
};

// 使用宏定义枚举与字符串的映射
NLOHMANN_JSON_SERIALIZE_ENUM(Color, {
    {Color::Unknown, nullptr},   // 默认值（可选）
    {Color::Red, "red"},
    {Color::Green, "green"},
    {Color::Blue, "blue"}
})

//方法二：手动实现 to_json / from_json
enum class Status {
    Pending,
    Active,
    Completed,
    Failed
};

// 枚举 → JSON（转为字符串）
void to_json(json& j, const Status& s) {
    switch (s) {
        case Status::Pending:   j = "pending"; break;
        case Status::Active:    j = "active"; break;
        case Status::Completed: j = "completed"; break;
        case Status::Failed:    j = "failed"; break;
        default:                j = "unknown"; break;
    }
}

// JSON → 枚举
void from_json(const json& j, Status& s) {
    std::string str = j.get<std::string>();
    if (str == "pending")        s = Status::Pending;
    else if (str == "active")    s = Status::Active;
    else if (str == "completed") s = Status::Completed;
    else if (str == "failed")    s = Status::Failed;
    else throw std::runtime_error("Unknown status: " + str);
}


// ============================================================
// 1. 自定义 struct
// ============================================================
// 方法一：使用宏（推荐，最简洁）
struct Person {
    std::string name;
    int age;
    double salary;
};

NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(Person, name, age, salary)


// 方法二：手动实现
struct Employee {
    std::string id;
    std::string department;
    bool active;
};

void to_json(json& j, const Employee& e) {
    j["id"] = e.id;
    j["department"] = e.department;
    j["is_active"] = e.active;
}

void from_json(const json& j, Employee& e) {
    e.id = j["id"];                    
    e.department = j["department"];   
    e.active = j["is_active"];        
}

// 方法三：在结构体中使用枚举
enum class Role { Admin, User, Guest };

NLOHMANN_JSON_SERIALIZE_ENUM(Role, {
    {Role::Admin, "admin"},
    {Role::User, "user"},
    {Role::Guest, "guest"}
})

struct Account {
    std::string username;
    Role role;              // 枚举成员
    bool active;
};

NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(Account, username, role, active)

// ============================================================
// 2. 自定义 class（私有成员）
// ============================================================

// 方法一：使用侵入式宏（INTRUSIVE）
class Product {
private:
    std::string sku;
    std::string name;
    double price;
    int stock;

public:
    Product() = default;
    ......

	// 侵入式宏：可访问私有成员
    NLOHMANN_DEFINE_TYPE_INTRUSIVE(Product, sku, name, price, stock)
};

// 方法二：使用友元函数
class BankAccount {
private:
    std::string account_number;
    std::string holder_name;
    double balance;
    bool is_active;

public:
    BankAccount() = default;
    ......

	// 声明友元函数
    friend void to_json(json& j, const BankAccount& acc);
    friend void from_json(const json& j, BankAccount& acc);
};

// 友元函数实现
void to_json(json& j, const BankAccount& acc) {
    j["account_number"] = acc.account_number;
    j["holder_name"] = acc.holder_name;
    j["balance"] = acc.balance;
    j["is_active"] = acc.is_active;
}

void from_json(const json& j, BankAccount& acc) {
    acc.account_number = j["account_number"]; 
    acc.holder_name = j["holder_name"];      
    acc.balance = j["balance"];              
    acc.is_active = j["is_active"];        
}

// 方法三：通过 Getter/Setter
class Rectangle {
private:
    double width;
    double height;
    std::string color;

public:
    Rectangle() : width(0), height(0), color("white") {}

	// Getter
    double getWidth() const { return width; }
    double getHeight() const { return height; }
    const std::string& getColor() const { return color; }
    double getArea() const { return width * height; }

    // Setter
    void setWidth(double w) { width = w; }
    void setHeight(double h) { height = h; }
    void setColor(const std::string& c) { color = c; }
};

// 通过 Getter/Setter 实现（类外部）
void to_json(json& j, const Rectangle& r) {
    j["width"] = r.getWidth();
    j["height"] = r.getHeight();
    j["color"] = r.getColor();
    j["area"] = r.getArea();
}

void from_json(const json& j, Rectangle& r) {
    r.setWidth(j["width"]); 
    r.setHeight(j["height"]); 
    r.setColor(j["color"]);
}

// 方法四：包含嵌套对象的类
class Order {
private:
    int order_id;
    Person customer;	 			// 嵌套 struct
    std::vector<Product> items;		// 嵌套 class 数组
    double total_amount;

public:
    Order() : order_id(0), total_amount(0) {}
    ......

	/*侵入式宏，前提：
		- 要么也使用 NLOHMANN_DEFINE_TYPE_INTRUSIVE 宏
		- 要么实现自定义的 to_json/from_json 函数
		- 要么使用其他 nlohmann json 支持的序列化方式*/
    NLOHMANN_DEFINE_TYPE_INTRUSIVE(Order, order_id, customer, items, total_amount)
};

// 方法五：带继承的类
//父类
class Shape {
protected:
    std::string type;
    std::string color;

public:
    Shape() = default;
    Shape(std::string t, std::string c) : type(std::move(t)), color(std::move(c)) {}
    virtual ~Shape() = default;

    const std::string& getType() const { return type; }
    const std::string& getColor() const { return color; }

    friend void to_json(json& j, const Shape& s);
    friend void from_json(const json& j, Shape& s);
};

void to_json(json& j, const Shape& s) {
    j["type"] = s.type;
    j["color"] = s.color;
}

void from_json(const json& j, Shape& s) {
    s.type = j["type"];
    s.color = j["color"];
}

//子类
class Circle : public Shape {
private:
    double radius;

public:
    Circle() : Shape("circle", "white"), radius(0) {}
    Circle(double r, std::string c) : Shape("circle", std::move(c)), radius(r) {}

    double getRadius() const { return radius; }
    double getArea() const { return 3.14159 * radius * radius; }

    friend void to_json(json& j, const Circle& c);
    friend void from_json(const json& j, Circle& c);
};

void to_json(json& j, const Circle& c) {
    j["type"] = c.type;
    j["color"] = c.color;
    j["radius"] = c.radius;
    j["area"] = c.getArea();
}

void from_json(const json& j, Circle& c) {
    c.type = j["type"];
    c.color = j["color"];
    c.radius = j["radius"]; 
}

// ============================================================
// 3. 裸指针 T*
// ============================================================
template<typename T>
void to_json(json& j, const T* ptr) {
    j = ptr ? json(*ptr) : json(nullptr);
}

template<typename T>
void from_json(const json& j, T*& ptr) {
    if (j.is_null()) {
        ptr = nullptr;
    } else {
        ptr = new T(j.get<T>());
    }
}

// ============================================================
// 4. std::shared_ptr<T>
// ============================================================
template<typename T>
void to_json(json& j, const std::shared_ptr<T>& ptr) {
    j = ptr ? json(*ptr) : json(nullptr);
}

template<typename T>
void from_json(const json& j, std::shared_ptr<T>& ptr) {
    ptr = j.is_null() ? nullptr : std::make_shared<T>(j.get<T>());
}

// ============================================================
// 5. std::unique_ptr<T>
// ============================================================
template<typename T>
void to_json(json& j, const std::unique_ptr<T>& ptr) {
    j = ptr ? json(*ptr) : json(nullptr);
}

template<typename T>
void from_json(const json& j, std::unique_ptr<T>& ptr) {
    ptr = j.is_null() ? nullptr : std::make_unique<T>(j.get<T>());
}

// ============================================================
// 6. std::variant<...>
// ============================================================
using MyVariant = std::variant<int, std::string, double>;

void to_json(json& j, const MyVariant& v) {
    std::visit([&j, &v](auto&& val) {
        j["type_index"] = v.index();
        j["value"] = val;
    }, v);
}

void from_json(const json& j, MyVariant& v) {
    size_t idx = j["type_index"];  // 直接赋值
    switch (idx) {
        case 0: v = static_cast<int>(j["value"]); break;
        case 1: v = static_cast<std::string>(j["value"]); break;
        case 2: v = static_cast<double>(j["value"]); break;
        default: throw std::runtime_error("Unknown variant type");
    }
}

// ============================================================
// 7. std::chrono::* 时间类型
// ============================================================
using TimePoint = std::chrono::system_clock::time_point;
using Duration = std::chrono::milliseconds;

void to_json(json& j, const TimePoint& tp) {
    j = std::chrono::duration_cast<Duration>(tp.time_since_epoch()).count();
}

void from_json(const json& j, TimePoint& tp) {
    int64_t ms = j;  // 直接赋值
    tp = TimePoint(Duration(ms));
}

void to_json(json& j, const Duration& d) {
    j = d.count();
}

void from_json(const json& j, Duration& d) {
    int64_t ms = j;  // 直接赋值
    d = Duration(ms);
}

// ============================================================
// 8. std::stack<T> / std::queue<T>
// ============================================================
template<typename T>
void to_json(json& j, std::stack<T> s) {
    std::vector<T> vec;
    while (!s.empty()) {
        vec.push_back(s.top());
        s.pop();
    }
    std::reverse(vec.begin(), vec.end());
    j = vec;
}

template<typename T>
void from_json(const json& j, std::stack<T>& s) {
    while (!s.empty()) s.pop();
    for (const auto& item : j) {
        T val = item;
        s.push(val);
    }
}

template<typename T>
void to_json(json& j, std::queue<T> q) {
    std::vector<T> vec;
    while (!q.empty()) {
        vec.push_back(q.front());
        q.pop();
    }
    j = vec;
}

template<typename T>
void from_json(const json& j, std::queue<T>& q) {
    while (!q.empty()) q.pop();
    for (const auto& item : j) {
        T val = item;
        q.push(val);
    }
}
```

## 3.从文件读取/写入 JSON
json重载了“<<”、“>>”运算符，所以可以直接从json读取/写入json
```cpp
// 写入 JSON 到文件
json js = {{"name", "赵六"}, {"age", 35}};
std::ofstream file_out("data.json");
file_out << js.dump(4); // 格式化写入
file_out.close();

// 从文件读取 JSON
std::ifstream file_in("data.json");
json js_read;
file_in >> js_read;
std::cout << "从文件读取：" << js_read["name"] << "\n";
```

## 4.json的核心函数
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260115225628592.png)

**注意事项**：
- **异常处理**：
	- json::parse()、at() 等操作可能抛异常（如格式错误、键不存在），**必须用 try-catch 捕获，避免程序崩溃**。
- **类型匹配**：
	- 读取值时要确保类型一致（如 JSON 是数字，不能直接转字符串），否则**会抛 type_error**
- **性能考量**：
	- nlohmann/json 易用但性能略低于 RapidJSON，若需极致性能（如高频解析/生成），可替换为 RapidJSON。
- **编码问题**：
	- 默认支持 UTF-8，若处理 GBK 等编码，需先转换为 UTF-8 再解析。


## 5.序列化与反序列化
- **序列化**：
	- C++类型先转为json，json在调用dump函数进行序列化，返回string
- **反序列化**：
	- 对于json序列化后的string，通过parse进行反序列化，得到C++类型

### （1）class的序列化与反序列化
```cpp
#include <nlohmann/json.hpp>
#include <vector>
#include <string>

using json = nlohmann::json;

// 使用侵入式宏的类
class User {
private:
    int id;
    std::string username;
    std::string email;
    std::vector<std::string> roles;

public:
    User() : id(0) {}  // 默认构造函数必须存在
    User(int id, std::string username, std::string email)
        : id(id), username(username), email(email) {}
        
    void addRole(const std::string& role) { roles.push_back(role); }
    
    // Getter（用于验证反序列化结果）
    int getId() const { return id; }
    const std::string& getUsername() const { return username; }
    const std::vector<std::string>& getRoles() const { return roles; }

    // 使用宏简化序列化/反序列化定义
    NLOHMANN_DEFINE_TYPE_INTRUSIVE(User, id, username, email, roles)
};

int main() {
    // 创建对象
    User user(1, "johndoe", "john@example.com");
    user.addRole("admin");
    user.addRole("user");
    
    // 序列化
    json j = user;
    std::string serialized = j.dump();
    
    // 反序列化
    User restoredUser = json::parse(serialized);
    
    return 0;
}


```

### （2）struct的序列化与反序列化
```cpp
#include <iostream>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

// 简单结构体
struct Person {
    std::string name;
    int age;
    bool active;
};

// 非侵入式序列化 - 在结构体外部定义转换
void to_json(json& j, const Person& p) {
    j = json{{"name", p.name}, {"age", p.age}, {"active", p.active}};
}

void from_json(const json& j, Person& p) {
    p.name = j["name"];
    p.age = j["age"];
    p.active = j["active"];
}

int main() {
    // 序列化
    Person p1{"Alice", 30, true};
    json j = p1;
    std::string serialized = j.dump(4);  // 4是缩进空格数
    std::cout << "序列化结果:\n" << serialized << std::endl;
    
    // 反序列化
    Person p2;
    json j2 = json::parse(serialized);
    from_json(j2, p2);
    // 或简写为: p2 = j2.get<Person>();
    
    std::cout << "反序列化后: " << p2.name << ", " << p2.age << std::endl;
    
    return 0;
}
```

### （3）文件读写示例（可用于“配置文件”）
```cpp
#include <nlohmann/json.hpp>
#include <fstream>
#include <iostream>

using json = nlohmann::json;

struct Config {
    std::string host;
    int port;
    bool debug;
};

NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(Config, host, port, debug)

int main() {
    // ========== 写入文件 ==========
    Config config{"127.0.0.1", 8080, true};
    
    std::ofstream ofs("config.json");
    if (ofs.is_open()) {
        json j = config;
        ofs << j.dump(4);  // 4 空格缩进
        ofs.close();
        std::cout << "配置已保存到 config.json" << std::endl;
    }
    
    // ========== 从文件读取 ==========
    std::ifstream ifs("config.json");
    if (ifs.is_open()) {
        json j = json::parse(ifs);
        Config loaded = j;
        ifs.close();
        
        std::cout << "读取配置: " << loaded.host << ":" << loaded.port 
                  << " (debug=" << std::boolalpha << loaded.debug << ")" << std::endl;
    }
    
    // ========== 异常处理 ==========
    try {
        std::ifstream bad_file("not_exist.json");
        json j = json::parse(bad_file);  // 会抛异常
    } catch (const json::parse_error& e) {
        std::cerr << "解析错误: " << e.what() << std::endl;
    }
    
    return 0;
}
```


## 补充：指针类型序列化 信息损失问题
上面演示的指针的序列化与反序列化明显有很大问题，会造成信息的损失
- **指针的序列化**：
	- 先转换为实体,然后进行序列化
- **指针的反序列化**：
	- 先获取实体，再转为指针
- 看个简单的例子：
	- 在植物大战僵尸中，需要实现存档功能，植物A持有僵尸B的指针，用来实现A攻击B，A、B在存档时都要进行序列化
		- 序列化前，A的指针执行B，序列化后进行存储；
		- 但反序列化后，A的指针指向的是用B的数据新生成的对象，显然A已经不指向B了，A也就无法攻击到B了


**解决方案**：
个典型的解决方案是使用对象引用系统或对象ID映射:
```cpp
// 基于ID的对象引用系统
class GameObject {
private:
    int id;  // 每个对象的唯一标识符
    
public:
    int getId() const { return id; }
    static GameObject* findById(int id); // 全局查找对象的方法
};

class Plant : public GameObject {
private:
    int zombieTargetId; // 存储目标僵尸的ID而非指针
    Zombie* zombieTarget; // 运行时使用的指针
    
public:
    void serialize(json& j) {
        j["id"] = getId();
        j["zombieTargetId"] = zombieTarget ? zombieTarget->getId() : -1;
        // 其他属性...
    }
    
    void deserialize(const json& j) {
        // 基本属性反序列化
        zombieTargetId = j["zombieTargetId"];
        // 指针暂时不设置，等所有对象都加载完成后再解析引用关系
    }
    
    // 解析阶段二：重建对象引用关系
    void resolveReferences() {
        if (zombieTargetId != -1) {
            zombieTarget = dynamic_cast<Zombie*>(GameObject::findById(zombieTargetId));
        }
    }
};
```

**实例**：
```cpp
#include <iostream>
#include <nlohmann/json.hpp>
#include <map>
#include <memory>
#include <vector>

using json = nlohmann::json;

// ==================== 通用的对象管理器 ====================

template<typename T>
class ObjectManager {
private:
    std::map<int, std::shared_ptr<T>> objects;
    int nextId = 1;

public:
    // 注册对象，返回 ID
    int registerObject(std::shared_ptr<T> obj) {
        int id = nextId++;
        objects[id] = obj;
        return id;
    }
    
    // 根据 ID 查找对象
    std::shared_ptr<T> findById(int id) {
        auto it = objects.find(id);
        return (it != objects.end()) ? it->second : nullptr;
    }
    
    // 获取所有对象
    const std::map<int, std::shared_ptr<T>>& getAll() const {
        return objects;
    }
    
    // 清空
    void clear() {
        objects.clear();
        nextId = 1;
    }
};

// ==================== 示例：植物大战僵尸 ====================

class Zombie;  // 前向声明

class Plant {
private:
    int id;
    std::string name;
    int health;
    int targetZombieId;  // 存储目标 ID 而非指针
    
    // 运行时指针（不序列化）
    std::shared_ptr<Zombie> targetZombie;

public:
    Plant() : id(0), health(100), targetZombieId(-1) {}
    Plant(int id, std::string name, int hp) 
        : id(id), name(std::move(name)), health(hp), targetZombieId(-1) {}
    
    int getId() const { return id; }
    void setId(int i) { id = i; }
    
    void setTarget(int zombieId) { targetZombieId = zombieId; }
    int getTargetId() const { return targetZombieId; }
    
    void setTargetPtr(std::shared_ptr<Zombie> z) { targetZombie = z; }
    std::shared_ptr<Zombie> getTargetPtr() const { return targetZombie; }
    
    const std::string& getName() const { return name; }
    
    // 序列化
    friend void to_json(json& j, const Plant& p) {
        j["id"] = p.id;
        j["name"] = p.name;
        j["health"] = p.health;
        j["target_zombie_id"] = p.targetZombieId;
    }
    
    // 反序列化
    friend void from_json(const json& j, Plant& p) {
        p.id = j["id"];
        p.name = j["name"];
        p.health = j["health"];
        p.targetZombieId = j["target_zombie_id"];
        // targetZombie 指针需要在解析阶段二设置
    }
};

class Zombie {
private:
    int id;
    std::string type;
    int health;

public:
    Zombie() : id(0), health(200) {}
    Zombie(int id, std::string type, int hp) 
        : id(id), type(std::move(type)), health(hp) {}
    
    int getId() const { return id; }
    void setId(int i) { id = i; }
    const std::string& getType() const { return type; }
    
    friend void to_json(json& j, const Zombie& z) {
        j["id"] = z.id;
        j["type"] = z.type;
        j["health"] = z.health;
    }
    
    friend void from_json(const json& j, Zombie& z) {
        z.id = j["id"];
        z.type = j["type"];
        z.health = j["health"];
    }
};

// ==================== 游戏存档管理器 ====================

class GameSaveManager {
private:
    ObjectManager<Plant> plantManager;
    ObjectManager<Zombie> zombieManager;

public:
    // 添加对象
    int addPlant(std::shared_ptr<Plant> p) {
        int id = plantManager.registerObject(p);
        p->setId(id);
        return id;
    }
    
    int addZombie(std::shared_ptr<Zombie> z) {
        int id = zombieManager.registerObject(z);
        z->setId(id);
        return id;
    }
    
    // 设置植物的攻击目标
    void setPlantTarget(int plantId, int zombieId) {
        auto plant = plantManager.findById(plantId);
        auto zombie = zombieManager.findById(zombieId);
        if (plant && zombie) {
            plant->setTarget(zombieId);
            plant->setTargetPtr(zombie);
        }
    }
    
    // 序列化整个游戏状态
    json serialize() {
        json j;
        
        // 序列化所有僵尸
        j["zombies"] = json::array();
        for (const auto& [id, zombie] : zombieManager.getAll()) {
            j["zombies"].push_back(*zombie);
        }
        
        // 序列化所有植物
        j["plants"] = json::array();
        for (const auto& [id, plant] : plantManager.getAll()) {
            j["plants"].push_back(*plant);
        }
        
        return j;
    }
    
    // 反序列化
    void deserialize(const json& j) {
        // 清空现有数据
        plantManager.clear();
        zombieManager.clear();
        
        // 阶段一：创建所有对象
        
        // 加载僵尸
        for (const auto& zj : j["zombies"]) {
            auto zombie = std::make_shared<Zombie>(zj.get<Zombie>());
            zombieManager.registerObject(zombie);
        }
        
        // 加载植物
        for (const auto& pj : j["plants"]) {
            auto plant = std::make_shared<Plant>(pj.get<Plant>());
            plantManager.registerObject(plant);
        }
        
        // 阶段二：重建引用关系
        for (const auto& [id, plant] : plantManager.getAll()) {
            int targetId = plant->getTargetId();
            if (targetId != -1) {
                auto zombie = zombieManager.findById(targetId);
                plant->setTargetPtr(zombie);
            }
        }
    }
    
    // 打印状态
    void printStatus() {
        std::cout << "=== 游戏状态 ===" << std::endl;
        for (const auto& [id, plant] : plantManager.getAll()) {
            std::cout << "植物 [" << plant->getName() << "] (ID:" << id << ")";
            if (auto target = plant->getTargetPtr()) {
                std::cout << " → 攻击目标: " << target->getType() 
                          << " (ID:" << target->getId() << ")";
            }
            std::cout << std::endl;
        }
        for (const auto& [id, zombie] : zombieManager.getAll()) {
            std::cout << "僵尸 [" << zombie->getType() << "] (ID:" << id << ")" << std::endl;
        }
    }
};

int main() {
    GameSaveManager game;
    
    // 创建游戏对象
    auto peashooter = std::make_shared<Plant>(0, "豌豆射手", 100);
    auto sunflower = std::make_shared<Plant>(0, "向日葵", 80);
    auto zombie1 = std::make_shared<Zombie>(0, "普通僵尸", 200);
    auto zombie2 = std::make_shared<Zombie>(0, "路障僵尸", 300);
    
    // 注册到管理器
    int p1 = game.addPlant(peashooter);
    int p2 = game.addPlant(sunflower);
    int z1 = game.addZombie(zombie1);
    int z2 = game.addZombie(zombie2);
    
    // 设置攻击关系
    game.setPlantTarget(p1, z1);  // 豌豆射手攻击普通僵尸
    
    std::cout << "========== 保存前 ==========" << std::endl;
    game.printStatus();
    
    // 序列化
    json save = game.serialize();
    std::cout << "\n========== JSON 存档 ==========\n" << save.dump(2) << std::endl;
    
    // 模拟游戏重启：创建新的管理器并加载存档
    GameSaveManager newGame;
    newGame.deserialize(save);
    
    std::cout << "\n========== 加载后 ==========" << std::endl;
    newGame.printStatus();
    
    return 0;
}
```
```cpp
========== 保存前 ==========
=== 游戏状态 ===
植物 [豌豆射手] (ID:1) → 攻击目标: 普通僵尸 (ID:1)
植物 [向日葵] (ID:2)
僵尸 [普通僵尸] (ID:1)
僵尸 [路障僵尸] (ID:2)

========== JSON 存档 ==========
{
  "plants": [
    {"health": 100, "id": 1, "name": "豌豆射手", "target_zombie_id": 1},
    {"health": 80, "id": 2, "name": "向日葵", "target_zombie_id": -1}
  ],
  "zombies": [
    {"health": 200, "id": 1, "type": "普通僵尸"},
    {"health": 300, "id": 2, "type": "路障僵尸"}
  ]
}


========== 加载后 ==========
=== 游戏状态 ===
植物 [豌豆射手] (ID:1) → 攻击目标: 普通僵尸 (ID:1)
植物 [向日葵] (ID:2)
僵尸 [普通僵尸] (ID:1)
僵尸 [路障僵尸] (ID:2)
```

## 快速参考表
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260116000619101.png)