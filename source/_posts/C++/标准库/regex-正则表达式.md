---
title: < regex > “正则表达式库”
date: 2026-01-27 22:34:17
tags:
	- 笔记
categories:
    - C++
    - 标准库
    - < regex > “正则表达式库”
---

# 1.概述
- C++11 正式引入 <regex> 标准库，提供正则表达式的匹配、查找、替换、分割等功能，底层基于 ECMAScript 正则语法（和 JavaScript/Perl 正则高度兼容），核心**解决文本模式匹配问题**，如：
	- **验证手机号、提取邮箱、格式化文本**等
```cpp
#include <regex>   // 核心头文件
```

# 2.核心组件
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129090604832.png)

## （1）std::regex —— 正则表达式对象
### <1>伪代码
```cpp
regex 对象:
    - 存储编译后的正则表达式模式
    - 可指定匹配选项（忽略大小写、优化等）

创建方式:
    regex(模式字符串)
    regex(模式字符串, 选项标志)

主要选项:
    icase     → 忽略大小写
    optimize  → 优化匹配速度
    nosubs    → 不保存捕获组（提升性能）
```
```cpp
// ==================== 伪代码 ====================
class regex {
public:
    // 构造函数
    regex();                                    // 空正则
    regex(const char* pattern);                 // 从 C 字符串
    regex(const string& pattern);               // 从 std::string
    regex(pattern, flags);                      // 带选项

    // 赋值
    regex& operator=(const char* pattern);
    regex& assign(pattern, flags);

    // 信息获取
    unsigned mark_count() const;                // 返回捕获组数量
    flag_type flags() const;                    // 返回当前选项

    // 选项常量
    static constexpr flag_type icase;           // 忽略大小写
    static constexpr flag_type nosubs;          // 不保存捕获组
    static constexpr flag_type optimize;        // 优化匹配速度
    static constexpr flag_type ECMAScript;      // ECMAScript 语法（默认）
    static constexpr flag_type basic;           // POSIX 基本语法
    static constexpr flag_type extended;        // POSIX 扩展语法
};
```

### <2>使用示例
```cpp
#include <regex>
#include <iostream>

int main() {
    // 1. 基本创建
    std::regex r1(R"(\d+)");                     // 匹配数字
    std::regex r2("\\d+");                        // 等价，但需转义
    
    // 2. 带选项
    std::regex r3("hello", std::regex::icase);    // 忽略大小写
    std::regex r4(R"(\w+)", std::regex::optimize | std::regex::nosubs);
    
    // 3. 从 string 构造
    std::string pattern = R"([a-z]+@[a-z]+\.[a-z]+)";
    std::regex r5(pattern);
    
    // 4. 获取信息
    std::regex r6(R"((\d{4})-(\d{2})-(\d{2}))");
    std::cout << "捕获组数量: " << r6.mark_count() << "\n";  // 3
}
```

## （2）std::smatch / std::cmatch —— 匹配结果
### <1>伪代码
```cpp
match_results 对象:
    - 存储一次匹配的所有信息
    - smatch 用于 std::string
    - cmatch 用于 const char*

访问方式:
    match[0]      → 完整匹配
    match[1]      → 第 1 个捕获组
    match[n]      → 第 n 个捕获组
    match.str(n)  → 同上，返回 string
    
位置信息:
    match.position(n)  → 第 n 组的起始位置
    match.length(n)    → 第 n 组的长度
    
上下文:
    match.prefix()  → 匹配之前的内容
    match.suffix()  → 匹配之后的内容
    
状态:
    match.size()   → 匹配组数量（包括 [0]）
    match.empty()  → 是否为空
    match.ready()  → 是否已执行匹配
```
```cpp
// ==================== 伪代码 ====================
class smatch {  // 或 cmatch（用于 const char*）
public:
    // 大小和状态
    bool empty() const;                         // 是否无匹配
    size_t size() const;                        // 匹配组数量（含 [0]）
    bool ready() const;                         // 是否已执行过匹配

    // 访问匹配内容
    sub_match operator[](size_t n) const;       // 第 n 组（0=完整匹配）
    string str(size_t n = 0) const;             // 第 n 组的字符串
    
    // 位置和长度
    difference_type position(size_t n = 0) const;  // 第 n 组起始位置
    difference_type length(size_t n = 0) const;    // 第 n 组长度

    // 上下文
    sub_match prefix() const;                   // 匹配之前的内容
    sub_match suffix() const;                   // 匹配之后的内容

    // 迭代器
    iterator begin() const;
    iterator end() const;
};
```

### <2>使用示例
```cpp
#include <regex>
#include <iostream>

int main() {
    // ============ smatch（用于 std::string）============
    std::string text = "Date: 2024-01-26, Time: 14:30";
    std::regex pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    std::smatch sm;
    
    if (std::regex_search(text, sm, pattern)) {
        std::cout << "=== smatch 示例 ===\n";
        std::cout << "完整匹配: " << sm[0] << "\n";     // 2024-01-26
        std::cout << "年: " << sm[1] << "\n";           // 2024
        std::cout << "月: " << sm[2] << "\n";           // 01
        std::cout << "日: " << sm[3] << "\n";           // 26
        
        std::cout << "位置: " << sm.position(0) << "\n";  // 6
        std::cout << "长度: " << sm.length(0) << "\n";    // 10
        
        std::cout << "匹配前: [" << sm.prefix() << "]\n"; // [Date: ]
        std::cout << "匹配后: [" << sm.suffix() << "]\n"; // [, Time: 14:30]
        
        // 遍历所有组
        for (size_t i = 0; i < sm.size(); ++i) {
            std::cout << "sm[" << i << "] = " << sm[i] << "\n";
        }
    }
    
    // ============ cmatch（用于 const char*）============
    const char* ctext = "ID: ABC123";
    std::regex id_pattern(R"(([A-Z]+)(\d+))");
    std::cmatch cm;
    
    if (std::regex_search(ctext, cm, id_pattern)) {
        std::cout << "\n=== cmatch 示例 ===\n";
        std::cout << "完整: " << cm[0] << "\n";   // ABC123
        std::cout << "字母: " << cm[1] << "\n";   // ABC
        std::cout << "数字: " << cm[2] << "\n";   // 123
    }
}
```

## （3）std::regex_match —— 完全匹配
### <1>伪代码
```cpp
regex_match(字符串, 正则):
    如果 整个字符串 完全匹配 正则:
        返回 true
    否则:
        返回 false

regex_match(字符串, 匹配结果, 正则):
    如果 整个字符串 完全匹配 正则:
        将匹配信息存入 匹配结果
        返回 true
    否则:
        返回 false

注意: 必须整个字符串匹配，不是找子串！
    "abc123" 匹配 "\d+"  → false（因为有 abc）
    "123"    匹配 "\d+"  → true
```
```cpp
// ==================== 伪代码 ====================
// 形式1：仅判断
bool regex_match(const string& str, const regex& re);
bool regex_match(const char* str, const regex& re);

// 形式2：获取匹配结果
bool regex_match(const string& str, smatch& m, const regex& re);
bool regex_match(const char* str, cmatch& m, const regex& re);

// 语义：
// if (整个字符串 == 正则模式) {
//     填充匹配结果（如果提供）
//     return true;
// } else {
//     return false;
// }
```
### <2>使用示例
```cpp
#include <regex>
#include <iostream>

int main() {
    // 1. 简单判断
    std::string email = "user@example.com";
    std::regex email_pattern(R"(\w+@\w+\.\w+)");
    
    if (std::regex_match(email, email_pattern)) {
        std::cout << "✅ 有效邮箱\n";
    }
    
    // 2. 获取匹配结果
    std::string date = "2024-01-26";
    std::regex date_pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    std::smatch match;
    
    if (std::regex_match(date, match, date_pattern)) {
        std::cout << "年: " << match[1] << "\n";  // 2024
        std::cout << "月: " << match[2] << "\n";  // 01
        std::cout << "日: " << match[3] << "\n";  // 26
    }
    
    // 3. ❌ 不是完全匹配的例子
    std::string text = "abc123xyz";
    std::regex num_pattern(R"(\d+)");
    
    if (!std::regex_match(text, num_pattern)) {
        std::cout << "❌ 不是完全匹配（有 abc 和 xyz）\n";
    }
    
    // 4. const char* 版本（使用 cmatch）
    const char* phone = "123-456-7890";
    std::regex phone_pattern(R"(\d{3}-\d{3}-\d{4})");
    std::cmatch cm;
    
    if (std::regex_match(phone, cm, phone_pattern)) {
        std::cout << "✅ 有效电话: " << cm[0] << "\n";
    }
}
```

## （4）std::regex_search —— 搜索子串匹配
### <1>伪代码
```cpp
regex_search(字符串, 正则):
    在字符串中搜索第一个匹配的子串
    如果 找到:
        返回 true
    否则:
        返回 false

regex_search(字符串, 匹配结果, 正则):
    在字符串中搜索第一个匹配的子串
    如果 找到:
        将匹配信息存入 匹配结果
        返回 true
    否则:
        返回 false

与 regex_match 的区别:
    regex_match("abc123xyz", "\d+")  → false（不是完全匹配）
    regex_search("abc123xyz", "\d+") → true（找到子串 "123"）
```
```cpp
// ==================== 伪代码 ====================
// 形式1：仅判断
bool regex_search(const string& str, const regex& re);
bool regex_search(const char* str, const regex& re);

// 形式2：获取匹配结果
bool regex_search(const string& str, smatch& m, const regex& re);
bool regex_search(const char* str, cmatch& m, const regex& re);

// 形式3：迭代器范围
bool regex_search(iterator first, iterator last, smatch& m, const regex& re);

// 语义：
// for (每个可能的起始位置) {
//     if (从该位置开始能匹配) {
//         填充匹配结果（如果提供）
//         return true;  // 找到第一个就返回
//     }
// }
// return false;
```

### <2>使用示例
```cpp
#include <regex>
#include <iostream>

int main() {
    std::string text = "Contact: alice@example.com or call 123-456-7890";
    
    // 1. 简单搜索
    std::regex email_pattern(R"(\w+@\w+\.\w+)");
    if (std::regex_search(text, email_pattern)) {
        std::cout << "✅ 找到邮箱\n";
    }
    
    // 2. 获取匹配详情
    std::regex phone_pattern(R"((\d{3})-(\d{3})-(\d{4}))");
    std::smatch match;
    
    if (std::regex_search(text, match, phone_pattern)) {
        std::cout << "找到电话: " << match[0] << "\n";      // 123-456-7890
        std::cout << "  区号: " << match[1] << "\n";         // 123
        std::cout << "  前缀: " << match[2] << "\n";         // 456
        std::cout << "  后缀: " << match[3] << "\n";         // 7890
        std::cout << "  位置: " << match.position() << "\n"; // 40
        std::cout << "  匹配前: [" << match.prefix() << "]\n";
        std::cout << "  匹配后: [" << match.suffix() << "]\n";
    }
    
    // 3. 循环搜索所有匹配（手动方式）
    std::string numbers = "a1b22c333d4444";
    std::regex num_pattern(R"(\d+)");
    std::smatch m;
    std::string::const_iterator search_start = numbers.cbegin();
    
    std::cout << "\n循环搜索所有数字:\n";
    while (std::regex_search(search_start, numbers.cend(), m, num_pattern)) {
        std::cout << "  找到: " << m[0] << "\n";
        search_start = m.suffix().first;  // 从匹配后的位置继续搜索
    }
}
```

## （5）std::regex_replace —— 替换匹配内容
### <1>伪代码
```cpp
regex_replace(字符串, 正则, 替换内容):
    将所有匹配替换为指定内容
    返回替换后的新字符串

替换内容中的特殊标记:
    $0 或 $&  → 整个匹配
    $1, $2... → 第 n 个捕获组
    $`        → 匹配之前的内容
    $'        → 匹配之后的内容
    $$        → 字面量 $

选项:
    format_first_only → 只替换第一个匹配
    format_no_copy    → 只输出匹配部分
```
```cpp
// ==================== 伪代码 ====================
// 形式1：返回新字符串
string regex_replace(const string& str, const regex& re, const string& fmt);
string regex_replace(const string& str, const regex& re, const string& fmt, match_flag_type flags);

// 形式2：输出到迭代器
OutputIt regex_replace(OutputIt out, iterator first, iterator last, 
                       const regex& re, const string& fmt);

// 替换格式字符串中的特殊标记：
// $0 或 $&  → 整个匹配
// $1, $2... → 第 n 个捕获组
// $`        → 匹配之前的内容
// $'        → 匹配之后的内容
// $$        → 字面量 $

// 替换选项：
// format_default     → 默认（替换所有）
// format_first_only  → 只替换第一个
// format_no_copy     → 只输出替换部分
```
### <2>使用示例
```cpp
#include <regex>
#include <iostream>

int main() {
    // 1. 简单替换
    std::string text = "Hello 123 World 456";
    std::regex num_pattern(R"(\d+)");
    
    std::string result = std::regex_replace(text, num_pattern, "###");
    std::cout << result << "\n";  // "Hello ### World ###"
    
    // 2. 使用捕获组反向引用
    std::string date = "2024-01-26";
    std::regex date_pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    
    // 调整日期格式：YYYY-MM-DD → MM/DD/YYYY
    std::string us_date = std::regex_replace(date, date_pattern, "$2/$3/$1");
    std::cout << us_date << "\n";  // "01/26/2024"
    
    // 3. 使用 $& 引用整个匹配
    std::string words = "hello world";
    std::regex word_pattern(R"(\w+)");
    
    std::string bracketed = std::regex_replace(words, word_pattern, "[$&]");
    std::cout << bracketed << "\n";  // "[hello] [world]"
    
    // 4. 只替换第一个匹配
    std::string text2 = "aaa bbb ccc";
    std::string first_only = std::regex_replace(
        text2, word_pattern, "XXX",
        std::regex_constants::format_first_only
    );
    std::cout << first_only << "\n";  // "XXX bbb ccc"
    
    // 5. 实用例子：格式化电话号码
    std::string raw_phone = "1234567890";
    std::regex phone_digits(R"((\d{3})(\d{3})(\d{4}))");
    
    std::string formatted = std::regex_replace(raw_phone, phone_digits, "($1) $2-$3");
    std::cout << formatted << "\n";  // "(123) 456-7890"
    
    // 6. 移除多余空格
    std::string messy = "Hello    World   !";
    std::regex multi_space(R"(\s+)");
    
    std::string clean = std::regex_replace(messy, multi_space, " ");
    std::cout << clean << "\n";  // "Hello World !"
}
```

## （6）std::sregex_iterator —— 遍历所有匹配
### <1>伪代码
```cpp
sregex_iterator 遍历流程:
    1. 创建迭代器: begin = sregex_iterator(字符串开始, 字符串结束, 正则)
    2. 创建结束标记: end = sregex_iterator()  // 默认构造
    3. 循环: for (it = begin; it != end; ++it)
    4. 每次迭代: *it 返回一个 smatch 对象

类型对应:
    std::string   → std::sregex_iterator
    const char*   → std::cregex_iterator
```
```cpp
// ==================== 伪代码 ====================
class sregex_iterator {  // 或 cregex_iterator（用于 const char*）
public:
    // 构造
    sregex_iterator();  // 结束迭代器
    sregex_iterator(iterator first, iterator last, const regex& re);

    // 迭代
    sregex_iterator& operator++();      // 移动到下一个匹配
    smatch operator*() const;           // 返回当前匹配结果
    smatch* operator->() const;         // 访问当前匹配结果

    // 比较
    bool operator==(const sregex_iterator& other) const;
    bool operator!=(const sregex_iterator& other) const;
};

// 语义：
// begin = sregex_iterator(str.begin(), str.end(), re);
// end   = sregex_iterator();  // 默认构造 = 结束
// 
// while (begin != end) {
//     *begin 是当前的 smatch
//     ++begin 移动到下一个匹配
// }
```

### <2>使用示例
```cpp
#include <regex>
#include <iostream>
#include <string>
#include <vector>

int main() {
    // 1. 基本遍历
    std::string text = "Emails: alice@a.com, bob@b.org, carol@c.net";
    std::regex email_pattern(R"(\w+@\w+\.\w+)");
    
    auto begin = std::sregex_iterator(text.begin(), text.end(), email_pattern);
    auto end = std::sregex_iterator();
    
    std::cout << "找到 " << std::distance(begin, end) << " 个邮箱:\n";
    
    // 重新创建迭代器（distance 会消耗迭代器）
    for (auto it = std::sregex_iterator(text.begin(), text.end(), email_pattern);
         it != std::sregex_iterator(); ++it) {
        std::smatch match = *it;
        std::cout << "  " << match.str() 
                  << " (位置: " << match.position() << ")\n";
    }
    
    // 2. 收集所有匹配到 vector
    std::string numbers = "a1b22c333d4444e55555";
    std::regex num_pattern(R"(\d+)");
    
    std::vector<std::string> all_numbers;
    for (auto it = std::sregex_iterator(numbers.begin(), numbers.end(), num_pattern);
         it != std::sregex_iterator(); ++it) {
        all_numbers.push_back(it->str());
    }
    
    std::cout << "\n所有数字:\n";
    for (const auto& n : all_numbers) {
        std::cout << "  " << n << "\n";
    }
    
    // 3. 带捕获组的遍历
    std::string dates = "日期: 2024-01-26, 2024-02-14, 2024-12-25";
    std::regex date_pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    
    std::cout << "\n解析所有日期:\n";
    for (auto it = std::sregex_iterator(dates.begin(), dates.end(), date_pattern);
         it != std::sregex_iterator(); ++it) {
        std::cout << "  " << (*it)[0] << " → "
                  << "年=" << (*it)[1] << ", "
                  << "月=" << (*it)[2] << ", "
                  << "日=" << (*it)[3] << "\n";
    }
}
```

## （7）std::sregex_token_iterator —— 分割字符串 / 提取子匹配
### <1>伪代码
```cpp
sregex_token_iterator 两种模式:

模式1 - 提取匹配/捕获组（submatch ≥ 0）:
    sregex_token_iterator(开始, 结束, 正则, submatch)
    submatch = 0  → 提取完整匹配
    submatch = 1  → 提取第 1 个捕获组
    submatch = {1, 2} → 依次提取第 1、2 个捕获组

模式2 - 分割字符串（submatch = -1）:
    sregex_token_iterator(开始, 结束, 分隔符正则, -1)
    返回分隔符之间的内容（类似 split）
```
```cpp
// ==================== 伪代码 ====================
class sregex_token_iterator {
public:
    // 构造
    sregex_token_iterator();  // 结束迭代器
    
    // submatch 指定提取哪部分：
    //   0, 1, 2...  → 提取第 n 个捕获组（0=完整匹配）
    //   -1          → 提取不匹配的部分（分割模式）
    //   {1, 2}      → 依次提取多个组
    sregex_token_iterator(iterator first, iterator last, 
                          const regex& re, int submatch);
    sregex_token_iterator(iterator first, iterator last, 
                          const regex& re, initializer_list<int> submatches);

    // 迭代
    sregex_token_iterator& operator++();
    string operator*() const;           // 返回当前子串

    // 比较
    bool operator==(const sregex_token_iterator& other) const;
    bool operator!=(const sregex_token_iterator& other) const;
};

// 语义 - 提取模式（submatch >= 0）：
// for (每个匹配) {
//     yield match[submatch];
// }

// 语义 - 分割模式（submatch == -1）：
// for (每个分隔符匹配) {
//     yield 分隔符之前的内容;
// }
// yield 最后一段;
```
### <2>使用示例
```cpp
#include <regex>
#include <iostream>
#include <vector>
#include <string>

int main() {
    // ============ 模式1：提取匹配内容 ============
    
    // 1.1 提取完整匹配（submatch = 0）
    std::string text1 = "a1b22c333";
    std::regex num_pattern(R"(\d+)");
    
    std::cout << "提取所有数字:\n";
    std::sregex_token_iterator it1(text1.begin(), text1.end(), num_pattern, 0);
    std::sregex_token_iterator end;
    
    for (; it1 != end; ++it1) {
        std::cout << "  " << *it1 << "\n";
    }
    // 输出: 1, 22, 333
    
    // 1.2 提取特定捕获组
    std::string dates = "2024-01-26, 2024-02-14";
    std::regex date_pattern(R"((\d{4})-(\d{2})-(\d{2}))");
    
    // 只提取月份（第 2 个捕获组）
    std::cout << "\n只提取月份:\n";
    std::sregex_token_iterator it2(dates.begin(), dates.end(), date_pattern, 2);
    for (; it2 != end; ++it2) {
        std::cout << "  " << *it2 << "\n";
    }
    // 输出: 01, 02
    
    // 1.3 提取多个捕获组
    std::cout << "\n提取年和日:\n";
    std::sregex_token_iterator it3(dates.begin(), dates.end(), date_pattern, {1, 3});
    for (; it3 != end; ++it3) {
        std::cout << "  " << *it3 << "\n";
    }
    // 输出: 2024, 26, 2024, 14
    
    // ============ 模式2：分割字符串（submatch = -1）============
    
    // 2.1 基本分割
    std::string csv = "apple,banana,cherry,date";
    std::regex comma(R"(,)");
    
    std::cout << "\n按逗号分割:\n";
    std::sregex_token_iterator it4(csv.begin(), csv.end(), comma, -1);
    std::vector<std::string> fruits(it4, end);
    
    for (const auto& f : fruits) {
        std::cout << "  [" << f << "]\n";
    }
    // 输出: [apple], [banana], [cherry], [date]
    
    // 2.2 处理连续分隔符
    std::string text2 = "a,,b,,,c";
    std::sregex_token_iterator it5(text2.begin(), text2.end(), comma, -1);
    
    std::cout << "\n连续逗号的情况:\n";
    for (; it5 != end; ++it5) {
        std::cout << "  [" << *it5 << "]\n";
    }
    // 输出: [a], [], [b], [], [], [c]
    
    // 2.3 按多种分隔符分割
    std::string mixed = "hello world\tfoo\nbar";
    std::regex whitespace(R"(\s+)");
    
    std::cout << "\n按空白字符分割:\n";
    std::sregex_token_iterator it6(mixed.begin(), mixed.end(), whitespace, -1);
    for (; it6 != end; ++it6) {
        std::cout << "  [" << *it6 << "]\n";
    }
    // 输出: [hello], [world], [foo], [bar]
    
    // 2.4 实用函数：通用 split
    auto split = [](const std::string& str, const std::string& delim) {
        std::regex re(delim);
        std::sregex_token_iterator begin(str.begin(), str.end(), re, -1), end;
        return std::vector<std::string>(begin, end);
    };
    
    auto parts = split("2024/01/26", "/");
    std::cout << "\n使用 split 函数:\n";
    for (const auto& p : parts) {
        std::cout << "  " << p << "\n";
    }
}
```

# 3. 异常处理
```cpp
#include <regex>
#include <iostream>

int main() {
    try {
        // 无效的正则表达式会抛出 std::regex_error
        std::regex bad_pattern(R"([)");  // 未闭合的方括号
        
    } catch (const std::regex_error& e) {
        std::cout << "正则表达式错误: " << e.what() << "\n";
        std::cout << "错误代码: " << e.code() << "\n";
        
        // 错误代码对应
        switch (e.code()) {
            case std::regex_constants::error_bracket:
                std::cout << "方括号不匹配\n";
                break;
            case std::regex_constants::error_paren:
                std::cout << "圆括号不匹配\n";
                break;
            // ... 其他错误类型
        }
    }
}
```

# 4.性能分析
```bash
┌─────────────────────────────────────────────────────────┐
│                std::regex 性能总结                       │
├─────────────────────────────────────────────────────────┤
│  优点：                                                  │
│    ✓ 标准库，无需额外依赖                                │
│    ✓ 接口简洁易用                                        │
│    ✓ 适合复杂模式匹配                                    │
│                                                         │
│  缺点：                                                  │
│    ✗ 性能差（比手写慢 50-100 倍）                        │
│    ✗ 可能灾难性回溯                                      │
│    ✗ 编译开销大                                          │
│                                                         │
│  使用原则：                                              │
│    → 复杂模式、低频操作 → 用 regex                       │
│    → 简单模式、高频操作 → 用 string 方法或手写           │
│    → 性能关键场景 → 考虑 RE2/PCRE2                       │
└─────────────────────────────────────────────────────────┘
```
## （1）C++ 标准库 regex 很慢
```cpp
// 性能测试：匹配 100 万次简单数字模式
#include <regex>
#include <chrono>
#include <iostream>

int main() {
    std::string text = "The answer is 42";
    std::regex pattern(R"(\d+)");
    std::smatch match;
    
    auto start = std::chrono::high_resolution_clock::now();
    
    for (int i = 0; i < 1'000'000; ++i) {
        std::regex_search(text, match, pattern);
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    
    std::cout << "std::regex: " << ms << " ms\n";
}
```
**典型结果（不同方法对比）：**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129092900058.png)

## （2）为什么 std::regex 这么慢？
```cpp
1. 编译开销大
   - 每次构造 std::regex 都要编译正则表达式
   - 即使是简单模式也要完整解析

2. 运行时匹配慢
   - 使用回溯算法（NFA 模拟）
   - 某些模式会导致指数级回溯（灾难性回溯）

3. 内存分配多
   - smatch 每次匹配都可能分配内存
   - 大量临时对象创建销毁

4. 没有 JIT 编译
   - 不像 PCRE2/RE2 可以编译成机器码
```

## （3）灾难性回溯（Catastrophic Backtracking）
### <1>什么是灾难性回溯？
```cpp
#include <regex>
#include <iostream>
#include <chrono>

int main() {
    // ⚠️ 危险的正则模式：嵌套量词
    std::regex evil_pattern(R"((a+)+b)");
    
    // 测试不同长度的不匹配字符串
    for (int len = 10; len <= 30; len += 5) {
        std::string text(len, 'a');  // "aaaa...a"（没有 b，不会匹配）
        
        auto start = std::chrono::high_resolution_clock::now();
        bool matched = std::regex_match(text, evil_pattern);
        auto end = std::chrono::high_resolution_clock::now();
        
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << "长度 " << len << ": " << ms << " ms\n";
    }
}
```
**可能的输出（时间指数增长！）：**
```bash
长度 10: 1 ms
长度 15: 32 ms
长度 20: 1024 ms
长度 25: 32768 ms  ← 已经 32 秒！
长度 30: ...       ← 可能要等几分钟到几小时
```

### <2>容易触发回溯的模式
```cpp
// ❌ 危险模式（避免使用）
R"((a+)+)"          // 嵌套量词
R"((a|a)+)"         // 重叠选择
R"(a*a*b)"          // 多个相邻量词匹配相同字符
R"(.*.*)"           // 多个贪婪通配

// ✅ 安全替代
R"(a+)"             // 去掉嵌套
R"(a+)"             // 合并重叠
R"(a*b)"            // 合并量词
R"(.*)"             // 只用一个
```

## （4）性能优化策略
## <1>复用 regex 对象（最重要！）
- 频繁创建 std::regex 对象会有性能开销，建议**复用**（如全局 / 静态对象）
```cpp
// ❌ 极差：每次都编译
void bad_search(const std::vector<std::string>& items) {
    for (const auto& item : items) {
        std::regex pattern(R"(\d+)");  // 💥 每次循环都编译！
        if (std::regex_search(item, pattern)) {
            // ...
        }
    }
}

// ✅ 好：复用 regex 对象
void good_search(const std::vector<std::string>& items) {
    static const std::regex pattern(R"(\d+)");  // 只编译一次
    for (const auto& item : items) {
        if (std::regex_search(item, pattern)) {
            // ...
        }
    }
}

// ✅ 更好：类成员或全局常量
class Parser {
    static inline const std::regex number_pattern{R"(\d+)"};
    static inline const std::regex email_pattern{R"(\w+@\w+\.\w+)"};
public:
    bool has_number(const std::string& s) {
        return std::regex_search(s, number_pattern);
    }
};
```

## <2>简化正则表达式
- 复杂正则（如嵌套分组、贪婪匹配）会降低效率，优先简化正则逻辑
```cpp
// ❌ 复杂（慢）
std::regex complex(R"(^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$)");

// ✅ 简化（如果精确度要求不高）
std::regex simple(R"(\S+@\S+\.\S+)");

// ❌ 贪婪匹配整个字符串
std::regex greedy(R"(.*target.*)");

// ✅ 非贪婪 + 锚定
std::regex lazy(R"(.*?target)");

// ❌ 多层捕获组
std::regex nested(R"((((\d+)-(\d+))-(\d+)))");

// ✅ 扁平化
std::regex flat(R"((\d+)-(\d+)-(\d+))");
```

## <3>使用优化选项
```cpp
// 使用 optimize 标志
std::regex pattern(R"(\d{4}-\d{2}-\d{2})", 
                   std::regex::optimize);  // 编译时优化

// 如果不需要捕获组，使用 nosubs
std::regex pattern2(R"(\d+)", 
                    std::regex::nosubs | std::regex::optimize);

// 组合使用
std::regex pattern3(R"(hello)", 
                    std::regex::icase | std::regex::optimize | std::regex::nosubs);
```
- optimize 效果：
	- 增加编译时间（约 2-5 倍）
	- 减少匹配时间（约 1.5-3 倍）
	- 适合：编译一次，匹配多次的场景
- nosubs 效果：
	- 不记录捕获组内容
	- 减少内存分配
	- 适合：只需要判断是否匹配，不需要提取内容

## <4>预检查 + 快速路径
```cpp
#include <regex>
#include <string>

// ✅ 先用简单检查过滤，再用正则精确匹配
bool is_valid_email(const std::string& email) {
    // 快速路径：基本检查
    if (email.length() < 5) return false;           // 最短 a@b.c
    if (email.find('@') == std::string::npos) return false;
    if (email.find('.') == std::string::npos) return false;
    
    // 慢速路径：正则验证
    static const std::regex pattern(
        R"(^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$)",
        std::regex::optimize
    );
    return std::regex_match(email, pattern);
}

// ✅ 对于简单模式，完全避免 regex
bool contains_digits_fast(const std::string& s) {
    // 比 regex_search(s, R"(\d)") 快 50-100 倍
    for (char c : s) {
        if (c >= '0' && c <= '9') return true;
    }
    return false;
}
```

# 附录、 常用正则表达式语法（ECMAScript）
## （1）字符类
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129090131203.png)

## （2）量词
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129090157665.png)

## （3）锚点
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129090219928.png)

## （4）分组与反向引用
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260129090237887.png)