---
title: 1.mysql在cpp中的基本用法
date: 2026-01-07 14:21:29
tags:
	- 笔记
categories:
    - C++
    - 开发笔记
    - mysql
    - 1.mysql在cpp中的基本用法
---

# 1.MySQL C 常用API

**注：如果要详细了解相关结构体或细节，可以参考"MySQL 8.4 C API 开发者指南"**
[MySQL 8.4 C API 开发者指南](https://dev.mysqlserver.cn/doc/c-api/8.4/en/c-api-introduction.html)

## 1.创建并初始化MYSQL句柄
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107202349321.png)


## 2.设置连接配置参数（可选）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107202427224.png)

## 3.尝试连接数据库
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107202454744.png)

## 4.执行SQL操作
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108081201418.png)

## 5.释放资源
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107202723688.png)

**1.必须手动释放的资源**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107215343290.png)

**2.不需要手动释放的资源**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107215419114.png)

## 6.其它
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107230322030.png)

## 7.游标相关
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107233502220.png)

## 8.事务操作
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107213938986.png)

## 9.防SQL注入
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260108085257356.png)

**(1)缺点：**
- 1.API繁琐
- 2.结果获取更麻烦：预处理语句获取结果需要预先绑定输出缓冲区，代码量是普通查询的 2-3 倍。
- 3.大多数场景性能差异不大：预处理的性能优势主要在重复执行同一 SQL 时，单次查询差别很小。

**(2)优点：**
- 1.“批量插入同结构数据”性能明显提升
- 2.高频重复查询，SQL解析只做一次
- 3.二进制数据(BLOB),避免转义问题

**(3)使用场景**
- 1.实际开发中，使用 “ORM框架”、“数据库抽象层”、“参数化拼接+转义”较多，原生 mysql_stmt_*	使用较少
- 2.适用：批量插入、高频查询、BLOB 时再考虑


---
# 2.mysql在cpp中使用的基本流程
**1.创建并初始化MYSQL句柄**
- mysql_init(conn_)
- 需检查初始化是否成功
- 注意：初始化成功后就分配资源了，后续任意一步出问题，都需要通过 mysql_close释放资源

**2.设置配置参数(可选)**
- 设置连接超时： mysql_options(conn_, MYSQL_OPT_CONNECT_TIMEOUT, &seconds)
- 设置字符集：mysql_options(conn_, MYSQL_SET_CHARSET_NAME, "utf8mb4")
- 设置读超时： mysql_options(conn_, MYSQL_OPT_READ_TIMEOUT, &seconds)
- 设置写超时： mysql_options(conn_, MYSQL_OPT_WRITE_TIMEOUT, &seconds)
- 设置自动重连：mysql_options(conn_, MYSQL_OPT_RECONNECT, &reconnect)

mysql_options一般很少出错，这里就不考虑出错

**3.尝试连接数据库**
- mysql_real_connect
- 需检查是否连接成功,失败则mysql_close(conn_)

**4.执行SQL操作**
- mysql_query
- mysql_affected_rows
- mysql_store_result
- mysql_fetch_row
- ......

**5.关闭连接 + 销毁句柄（释放资源）**
- mysql_free_result
- mysql_close


# 3.封装示例
**由于本人目前能力有限，可能实现并不是很好，仅作API使用演示**

## 1.连接封装
```cpp
//mysql_connection.h

#include "config/config.h"
#include "mysql_result.h"
#include <mysql/mysql.h>
#include <string>
#include <variant>
namespace user{



class MySQLConnection{
public:
    using Param=std::variant<nullptr_t,int64_t,uint64_t,double,std::string>;

    explicit MySQLConnection(const MySQLConfig& config);
    ~MySQLConnection();

    //禁止拷贝，允许移动
    MySQLConnection(const MySQLConnection&)=delete;
    MySQLConnection& operator=(const MySQLConnection&)=delete;
    MySQLConnection(MySQLConnection&& other)=delete;
    MySQLConnection& operator=(MySQLConnection&& other)=delete;

    //有效性检查
    bool Valid(){return mysql_ && mysql_ping(mysql_)==0;}

    MYSQL* Get() {return mysql_;}

    //Select操作
    //存在限制：“sql中不可以包含非占位符'?'”
    MySQLResult Query(const std::string& sql,std::initializer_list<Param> param={});

    //Insert/Update/Delete操作
    //存在限制：“sql中不可以包含非占位符'?'”
    uint64_t Execute(const std::string& sql,std::initializer_list<Param> params={});

    // 流式查询（逐行获取，适合大结果集）
    // 注意：遍历期间不能执行其他查询！
    // MySQLResult QueryUnbuffered(const std::string& sql);

    //获取Insert后的id
    uint64_t LastInsertId();

private:
    // 初始化并设置选项
    void InitAndSetOptions(const MySQLConfig& config);

     // 判断错误是否值得重试
    bool IsRetryableError(unsigned int err_code);

    //重试连接
    void ConnectWithRetry(const MySQLConfig& config);

    //转义方法（只能用于转义“参数值”）（防SQL注入的简单处理）
    std::string Escape(const std::string& str);

    //SQL语句包装器： “？为占位符 ——>sql语句中不可以包含非占位符 ？”
    std::string BuildSQL(const std::string& sql, std::initializer_list<Param> params);
private:
    MYSQL* mysql_=nullptr;
};

}
```

```cpp
//mysql_connection.cpp

#include "mysql_connection.h"
#include "common/logger.h"
#include "server/exception.h"
#include<chrono>
#include<vector>
namespace user{

MySQLConnection::MySQLConnection(const MySQLConfig& config){
    try{
        // 初始化并设置选项
        InitAndSetOptions(config);
    }catch(...){
        if(mysql_){
            mysql_close(mysql_);
            mysql_=nullptr;
        }
        throw;
    }

    if(!mysql_real_connect(mysql_,
                            config.host.c_str(),
                            config.username.c_str(),
                            config.password.c_str(),
                            config.database.c_str(),
                            config.port,
                            nullptr,
                            0)){
        //一定要先保存错误信息，否则释放mysql_后就无法获取了
        unsigned int err_code = mysql_errno(mysql_);
        std::string err_msg = mysql_error(mysql_);

        //连接失败
        mysql_close(mysql_);
        mysql_=nullptr;
        
        bool should_retry = config.auto_reconnect.value_or(false) && 
                    config.max_retries > 0 &&
                    IsRetryableError(err_code);

        if (!should_retry) {
            LOG_ERROR("mysql_real_connect failed [{}]: {}", err_code, err_msg);
            throw MySQLException(err_code, err_msg);
        }
        
        //重连
        ConnectWithRetry(config);
    }
}

MySQLConnection::~MySQLConnection(){
    if(mysql_){
        mysql_close(mysql_);
        mysql_=nullptr;
    }
}




MySQLResult MySQLConnection::Query(const std::string& sql,std::initializer_list<Param> params){
    std::string new_sql=BuildSQL(sql,params);
    
    //执行查询
    if (mysql_query(mysql_, new_sql.c_str()) != 0) {
        unsigned int err_code = mysql_errno(mysql_);
        std::string err_msg = mysql_error(mysql_);
        LOG_ERROR("Query failed [{}]: {} | SQL: {}", err_code, err_msg, new_sql);
        throw MySQLException(err_code, err_msg);
    }

    //获取结果集
    MYSQL_RES* res=mysql_store_result(mysql_);
    
    // 注意：SELECT 语句执行出错时， res为nullptr（空集时不会为nullptr）
    // 但如果是非 SELECT 语句调用了 Query，res 也会是 nullptr
    if (res == nullptr && mysql_field_count(mysql_) > 0) {
        // 应该有结果但没拿到，说明出错了
        unsigned int err_code = mysql_errno(mysql_);
        std::string err_msg = mysql_error(mysql_);
        LOG_ERROR("mysql_store_result failed [{}]: {}", err_code, err_msg);
        throw MySQLException(err_code, err_msg);
    }

    return MySQLResult(res);
}


uint64_t MySQLConnection::Execute(const std::string& sql,std::initializer_list<Param> params){
    std::string new_sql=BuildSQL(sql,params);
    //执行查询
    if (mysql_query(mysql_, new_sql.c_str()) != 0) {
        unsigned int err_code = mysql_errno(mysql_);
        std::string err_msg = mysql_error(mysql_);
        LOG_ERROR("Query failed [{}]: {} | SQL: {}", err_code, err_msg, new_sql);
        throw MySQLException(err_code, err_msg);
    }

    return mysql_affected_rows(mysql_);
}

//获取Insert后的id
uint64_t MySQLConnection::LastInsertId(){
    return mysql_insert_id(mysql_);
}

void MySQLConnection::InitAndSetOptions(const MySQLConfig& config){
    //初始化连接句柄
    mysql_=mysql_init(nullptr);
    if(!mysql_){
        LOG_ERROR("mysql_init failed");
        //注意：不可以throw MySQLException(mysql_errno(mysql_),mysql_error(mysql_));
        //因为mysql_ 是 nullptr，mysql_errno(nullptr) 行为未定义
        throw MySQLException(0, "mysql_init failed: out of memory");
    }

    //设置参数
    if(config.connection_timeout_ms.has_value()){
        int connection_timeout=config.connection_timeout_ms.value()/1000; //ms—>s
        mysql_options(mysql_,MYSQL_OPT_CONNECT_TIMEOUT,&connection_timeout);
    }
    if(config.read_timeout_ms.has_value()){
        int read_timeout=config.read_timeout_ms.value()/1000;
        mysql_options(mysql_,MYSQL_OPT_READ_TIMEOUT,&read_timeout);
    }
    if(config.write_timeout_ms.has_value()){
        int write_timeout=config.write_timeout_ms.value()/1000;
        mysql_options(mysql_,MYSQL_OPT_WRITE_TIMEOUT,&write_timeout);
    }
    if(config.auto_reconnect.has_value()){
        /*MySQL C API 底层不直接支持 bool 类型，而是用 unsigned int（无符号整数）来表示布尔值，
          0 代表 false，非 0 代表 true。*/
        unsigned int mybool=config.auto_reconnect.value()?1:0;
        mysql_options(mysql_,MYSQL_OPT_RECONNECT,&mybool);
    }

    mysql_options(mysql_,MYSQL_SET_CHARSET_NAME,config.charset.c_str());
}

// 判断错误是否值得重试
bool MySQLConnection::IsRetryableError(unsigned int err_code) {
    switch (err_code) {
        // 可重试的错误（网络/临时性问题）
        case 2002:  // CR_CONNECTION_ERROR - socket 错误
        case 2003:  // CR_CONN_HOST_ERROR - 无法连接主机
        case 2006:  // CR_SERVER_GONE_ERROR - 服务器断开
        case 2013:  // CR_SERVER_LOST - 连接丢失
            return true;
        
        // 不可重试的错误（配置/认证问题）
        case 1045:  // ER_ACCESS_DENIED_ERROR - 认证失败
        case 1044:  // ER_DBACCESS_DENIED_ERROR - 无权限
        case 1049:  // ER_BAD_DB_ERROR - 数据库不存在
        case 2005:  // CR_UNKNOWN_HOST - 未知主机
        default:
            return false;
    }
}

void MySQLConnection::ConnectWithRetry(const MySQLConfig& config){
    unsigned int err_code=0;
    std::string err_msg;
    
    for(unsigned int attempt=1;attempt<=config.max_retries;++attempt){
        try{
            InitAndSetOptions(config);
            //尝试连接
            if(mysql_real_connect(mysql_,
                                  config.host.c_str(),
                                  config.username.c_str(),
                                  config.password.c_str(),
                                  config.database.c_str(),
                                  config.port,
                                  nullptr,
                                  0)){
                // 连接成功
                LOG_INFO("MySQL connected to {}:{}/{} (attempt {}/{})",
                        config.host, config.port, config.database,
                        attempt,config.max_retries);
                return;
            }
    
            //连接失败
            err_code=mysql_errno(mysql_);
            err_msg=mysql_error(mysql_);

            //释放当前句柄
            mysql_close(mysql_);
            mysql_=nullptr;

            LOG_WARN("MySQL connect attempt {}/{} failed [{}]: {}",
                attempt, config.max_retries, err_code, err_msg);

            if(!IsRetryableError(err_code)){
                LOG_ERROR("Non-retryable error, giving up");
                break;
            }

            //尝试重连（等待一段时间）
            LOG_INFO("Retrying in {} ms...", config.retry_interval_ms);
            std::this_thread::sleep_for(std::chrono::milliseconds(config.retry_interval_ms));
        }catch(...){
            //捕捉InitAndSetOptions等抛出的异常
            if(mysql_){
                mysql_close(mysql_);
                mysql_=nullptr;
            }
            throw; //继续向上传递
        }
    }

    // 所有重试都失败
    LOG_ERROR("all Retry are failed. mysql connect failed");
    throw MySQLException(err_code, 
        "Failed to connect after " + std::to_string(config.max_retries) + 
        " attempts: " + err_msg);
}

//转义方法（防SQL注入的简单处理）
std::string MySQLConnection::Escape(const std::string& str){
    if(!mysql_){
        LOG_ERROR("Connection not established");
        throw MySQLException(0, "Connection not established");
    }

    //最坏情况“都需要转移”：所以将空间设置为 2*str.size()+1（官方推荐长度）
    //使用vector更安全，也避免手动管理空间
    std::vector<char> buffer(str.size()*2+1);
    unsigned long len=mysql_real_escape_string(mysql_,buffer.data(),str.c_str(),
                                                static_cast<unsigned long>(str.size()));
    
    
    return std::string(buffer.data(),len);
}

//SQL语句包装器
std::string MySQLConnection::BuildSQL(const std::string& sql, std::initializer_list<Param> params){
    // 预分配内存：初始容量为SQL长度 + 参数平均长度*参数数量（减少内存重分配）
    std::string result;
    result.reserve(sql.size() + params.size() * 32);

    auto param=params.begin();
    //解析sql中的'?'占位符，并用params中的参数填入
    for(auto& c:sql){
        if(c=='?'){
            if(param==params.end()){
                //参数不足
                LOG_ERROR("参数不足");
                throw std::runtime_error("param is not enough");
            }

            std::visit([&](auto&& val){
                using T=std::decay_t<decltype(val)>;
                if constexpr(std::is_same_v<T,nullptr_t>){
                    result+="NULL";
                }else if constexpr(std::is_same_v<T,std::string>){
                    //字符串要加“转义单引号”
                    result+="\'";
                    result+=Escape(val);  //进行转义：方sql注入
                    result+="\'";
                }else{
                    result+=std::to_string(val);
                }
            },*param);

            param++;  //参数后移
        }else{
            //普通字符
            result.push_back(c);
        }
    }
    if(param!=params.end()){
        LOG_ERROR("参数过多");
        throw std::runtime_error("too many param");
    }

    return result;
}

}

```

## 2.结果集封装
```cpp
//mysql_result.h

#pragma once
#include <mysql/mysql.h>
#include <string>
#include <cstdint>
#include <stdexcept>
#include <optional>
namespace user{

//管理结果集
class MySQLResult {
public:
    explicit MySQLResult(MYSQL_RES* res);
    ~MySQLResult();

    // 禁用拷贝，允许移动
    MySQLResult(const MySQLResult&) = delete;
    MySQLResult& operator=(const MySQLResult&) = delete;
    //移动操作加 noexcept 可以让 STL 容器更好地优化。
    MySQLResult(MySQLResult&& other) noexcept;
    MySQLResult& operator=(MySQLResult&& other) noexcept;

    // 遍历：获取下一行存储在current_row_中，无数据则返回false
    bool Next();
    
    // 元信息
    size_t RowCount() const;
    size_t FieldCount() const;
    
    // 取值（列索引从 0 开始）
    bool IsNull(size_t col) const;
    std::optional<std::string> GetString(size_t col) const;
    std::optional<int64_t> GetInt(size_t col) const;
    std::optional<double> GetDouble(size_t col) const;
    
    // 空结果集判断
    bool Empty() const { return result_ == nullptr || RowCount() == 0; }

private:
    //检查访问的列是否合法，不合法抛出异常
    void CheckColumn(size_t col) const;

    MYSQL_RES* result_ = nullptr;       //结果集
    MYSQL_ROW current_row_ = nullptr;   //当前行
    unsigned long* lengths_ = nullptr;  //当前行各列字段值的实际长度数组
    unsigned int field_count_ = 0;      //结果集的列数
};

}
```

```cpp
//mysql_result.cpp

#include "mysql_result.h"
#include "server/exception.h"
#include <stdexcept>
#include <cstdlib>
namespace user{
    
MySQLResult::MySQLResult(MYSQL_RES* res)
:result_(res),
    field_count_(res?mysql_num_fields(res):0)
{}

MySQLResult::~MySQLResult(){
    if(result_){
        mysql_free_result(result_);
        result_=nullptr;
    }
}

MySQLResult::MySQLResult(MySQLResult&& other) noexcept
    : result_(other.result_),
        current_row_(other.current_row_),
        lengths_(other.lengths_),
        field_count_(other.field_count_){
    other.result_=nullptr;
    other.current_row_=nullptr;
    other.lengths_=nullptr;
    other.field_count_=0;
}

MySQLResult& MySQLResult::operator=(MySQLResult&& other) noexcept{
    //注意检查&other!=this
    if(&other!=this){
        if(result_){
            //释放自己的资源
            mysql_free_result(result_);
        }
        result_=other.result_;
        current_row_=other.current_row_;
        lengths_=other.lengths_;
        field_count_=other.field_count_;

        other.result_=nullptr;
        other.current_row_=nullptr;
        other.lengths_=nullptr;
        other.field_count_=0;
    }
    return *this;
}

bool MySQLResult::Next(){
    if(!result_) return false;
    //获取一行数据（游标自动后移）
    current_row_=mysql_fetch_row(result_);
    if(current_row_){
        //获取各字段长度
        lengths_=mysql_fetch_lengths(result_);
        return true;
    }
    return false;
}

size_t MySQLResult::RowCount() const{
    return result_?static_cast<size_t>(mysql_num_rows(result_)):0;
}

size_t MySQLResult::FieldCount() const{
    return field_count_;
}

void MySQLResult::CheckColumn(size_t col) const{
    if(!current_row_){
        throw std::runtime_error("No current row, call Next() first");
    }
    else if(col>=field_count_){
        throw std::out_of_range("Column index " + std::to_string(col) + 
                            " out of range, max is " + std::to_string(field_count_ - 1));
    }
}

bool MySQLResult::IsNull(size_t col) const{
    CheckColumn(col);
    return current_row_[col]==nullptr;
}

std::optional<std::string> MySQLResult::GetString(size_t col) const{
    CheckColumn(col);
    //mysql中，可能存的是NULL
    if(current_row_[col]==nullptr){
        return std::nullopt;
    }
    //current_row_可能包含的是二进制，不可以直接返回，避免被“截断”
    if(lengths_){
        return std::string(current_row_[col],lengths_[col]);
    }
    return current_row_[col];
}

std::optional<int64_t> MySQLResult::GetInt(size_t col) const{
    CheckColumn(col);
    //mysql中，可能存的是NULL
    if(current_row_[col]==nullptr){
        return std::nullopt;
    } 
    return std::strtoll(current_row_[col],nullptr,10);
}

std::optional<double> MySQLResult::GetDouble(size_t col) const{
    CheckColumn(col);
    //mysql中，可能存的是NULL
    if(current_row_[col]==nullptr){
        return std::nullopt;
    }
    return std::strtod(current_row_[col],nullptr);
}

}
```