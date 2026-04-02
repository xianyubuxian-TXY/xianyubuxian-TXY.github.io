---
title: C++语法
date: 2026-03-20 22:21:59
tags:
	- 笔记
categories:
	- C++
	- C++语法
	- C++语法
---

# 一、语法错误示例

## 示例1
```cpp
class OrderBook{
public:
	// ......
private:
    using OrderPtr=std::shared_ptr<Order>;

    // OrderBook需要管理Order，补全你需要的成员    
    // 价格大顶堆，时间小顶堆
    auto less_cmp=[](OrderPtr& A,OrderPtr B){	// 	❌ 错误：类的非静态成员不能这样直接用 auto 做占位类型
        if(A->GetPrice()==B->GetPrice()) return A->getSymbol()>B->getSymbol();
        return A->GetPrice() < B->GetPrice();
    };

    // 价格小顶堆，时间小顶堆
    auto greater_cmp=[](OrderPtr& A,OrderPtr B){	// 	❌ 错误：类的非静态成员不能这样直接用 auto 做占位类型
        if(A->GetPrice()==B->GetPrice()) return A->getSymbol()>B->getSymbol();
        return A->GetPrice() > B->GetPrice();
    };

    // 大顶堆：存储买单
    std::priority_queue<OrderPtr,std::vector<OrderPtr>,decltype(less_cmp)> buyOrderPq_(less_cmp); //❌ 错误：类内成员默认初始化不能这么用 ()。这种写法很容易被编译器当成“函数声明”
    std::unordered_map<int,OrderPtr> buyMap_;

    // 小顶堆：存储买单
    std::priority_queue<OrderPtr,std::vector<OrderPtr>,decltype(greater_cmp)> sellOrderPq_(greater_cmp); // ❌ 错误：类内成员默认初始化不能这么用 ()。这种写法很容易被编译器当成“函数声明”
    std::unordered_map<int,OrderPtr> sellMap_;

    // 匹配订单数组{买单，卖单}
    std::vector<std::pair<OrderPtr,OrderPtr>> matchVec_;

};
```

### （1）错误1 与 改正
#### <1>错误示例
- **类的非静态成员不能这样直接用 auto 做占位类型**
```cpp
    auto less_cmp=[](OrderPtr& A,OrderPtr B){	// 	❌ 错误：类的非静态成员不能这样直接用 auto 做占位类型
        if(A->GetPrice()==B->GetPrice()) return A->getSymbol()>B->getSymbol();
        return A->GetPrice() < B->GetPrice();
    };

    // 价格小顶堆，时间小顶堆
    auto greater_cmp=[](OrderPtr& A,OrderPtr B){	// 	❌ 错误：类的非静态成员不能这样直接用 auto 做占位类型
        if(A->GetPrice()==B->GetPrice()) return A->getSymbol()>B->getSymbol();
        return A->GetPrice() > B->GetPrice();
    };
```

#### <2>改正方法
- 最稳妥的写法是改成“**比较器结构体**”
```cpp
    struct ButCmp{
        bool operator()(const OrderPtr& a,const OrderPtr& b) const {
            if(a->GetPrice() == b->GetPrice()){
                return a->getSymbol() > b->getSymbol();
            }
            return a->GetPrice() < b->GetPrice();
        }
    };

    struct SellCmp{
        bool operator()(const OrderPtr& a,const OrderPtr& b)const{
            if(a->GetPrice()==b->GetPrice()){
                return a->getSymbol() > b->getSymbol();
            }
            return a->GetPrice() > b->GetPrice();
        }

    };
```
- 放到类里时，用 **inline static auto**
```cpp
    inline static auto buy_cmp = [](const OrderPtr& a, const OrderPtr& b) {
        if (a->GetPrice() == b->GetPrice()) {
            return a->getSymbol() > b->getSymbol();
        }
        return a->GetPrice() < b->GetPrice();
    };

    inline static auto sell_cmp = [](const OrderPtr& a, const OrderPtr& b) {
        if (a->GetPrice() == b->GetPrice()) {
            return a->getSymbol() > b->getSymbol();
        }
        return a->GetPrice() > b->GetPrice();
    };
```

### （2）错误2
#### <1>错误示例
- **类内成员默认初始化不能这么用 ()**：这种写法很容易被编译器当成“函数声明”，所以调用 buyOrderPq_.push()、buyOrderPq_.empty() 时，编译器会说它像个函数，不像个对象。 
```cpp
std::priority_queue<OrderPtr,std::vector<OrderPtr>,decltype(less_cmp)> buyOrderPq_(less_cmp); // ❌ 错误

std::priority_queue<OrderPtr,std::vector<OrderPtr>,decltype(greater_cmp)> sellOrderPq_(greater_cmp); // ❌ 错误
```

#### <2>改正方法
- **使用{}初始化**
```cpp
std::priority_queue<OrderPtr,std::vector<OrderPtr>,decltype(less_cmp)> buyOrderPq_{less_cmp};

std::priority_queue<OrderPtr,std::vector<OrderPtr>,decltype(greater_cmp)> sellOrderPq_{greater_cmp};
```