---
title: 15.std::optional<>
date: 2026-01-07 19:04:20
tags:
	- 笔记
categories:
    - C++
    - 《C++17完全指南》
    - 三、新的标准库组件
    - 15.std::optional<>
---

# 1.概述
- **1.std::optional<>模拟了一个可以为空的任意类型的实例,可以被用作成员、参数、返回值等**
- **2.拷贝一个无内含值的std::optional<>的开销很小**
- **3.但拷贝有内含值的std::optional<>的开销约等于拷贝内含值的开销**
- **4.std::optional<>对象也支持move语义**

# 2.常用API
## 0.头文件
**#include\<optional\>**

## 1.创建/初始化
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107191619544.png)

## 2.取值相关
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107192204439.png)

## 3.状态判断
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107191715927.png)

## 4.修改/重置
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107191826919.png)

## 5.其它常用API
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260107191849790.png)

**注意：普通类型可以隐式转换为 std::optional,std::optional 不能直接隐式转换为普通类型**
```cpp
/*1.普通类型——>optional*/
std::optional<int> opt = 100; // 隐式转换（int → optional<int>）
std::optional<std::string> opt_str = "mysql"; // 隐式转换（const char* → string → optional<string>）

/*2.optional——>普通类型*/
std::optional<int> opt = 100;
// int val = opt; // 错误：不支持隐式转换

// 正确：显式取值
int val1 = *opt;          // 确定有值时用
int val2 = opt.value();   // 允许抛异常时用
int val3 = opt.value_or(0); // 空值兜底时用

```

# 3.使用示例
## 1.可选的返回值
```cpp
#include<optional>

std::optional<int> asInt(const std::string& s){
	try{
		size_t pos;
        int result = std::stoi(s, &pos);  // stoi 会抛出异常
        if (pos != s.length()) {  // 部分转换
            return std::nullopt;
        }
        return result;
	}catch(...){
		return std::nullopt;
	}
}

int main(){
	for(auto s: {"111","123c"}){
		std::optional<int> oi=asInt(s);
		if(oi.has_value()){ // 或 if(oi)
			std::cout<<"convert '"<<s<<"' to int: "<<oi.value()<<"\n";
		}else{
			std::cout<<"convert failed"<<"\n";
		}
	}
}
```

## 2.可选的参数和数据成员
```cpp
#include<optional>

class Name
{
private:
	std::string first;
	std::optional<std::string> middle;
	std::string last;
public:
	Name (std::string f, std::optional<std::string> m, std::string l)
		:first{std::move(f)}, middle{std::move(m)}, last{std::move(l)} { //移动构造
	}

	friend std::ostream& operator << (std::ostream& strm, const Name& n) {
		strm << n.first << ' ';
		if (n.middle.has_value()) { //也可以直接： n.middle
			strm << n.middle.value() << ' ';  //也可以： *n.middle
		}
		return strm << n.last;
	}
};
```



## 3.实际使用场景：连接池配置
### (1)概述
- **在配置中，有些配置项可能使用，也可能不使用，这时就可以std::optional<>**
- **配置是从文件中读取，然后根据有无值，存储到配置结构体中**

### (2)代码示例
```yaml
# mysql部分配置（yaml格式）
mysql:
  # ...
  # 超时配置（毫秒）
  connection_timeout_ms: 5000  	# 存在
  # read_timeout_ms: 30000		# 不存在
  # write_timeout_ms: 30000		# 不存在
  
  # 重试配置
  max_retries: 3
  retry_interval_ms: 1000
  
  # 其他配置
  auto_reconnect: true			# 存在
```

```cpp
//MySQL配置部分代码
struct MySQLConfig {
	...

    //超时配置(ms)(可能有，可能无)
    std::optional<unsigned int> connection_timeout_ms;
    std::optional<unsigned int> read_timeout_ms;
    std::optional<unsigned int> write_timeout_ms;
    
    //连接失败，重试配置
    unsigned int max_retries=3; 
    unsigned int retry_interval_ms=1000; //重试间隔（ms）

    //其它配置
    std::optional<bool> auto_reconnect; //可能有可能无
    ...
};

...
void MySQLConnection::InitAndSetOptions(const MySQLConfig& config){
    ...

    //如果“连接超时”存在，就设置“连接超时选项”
    if(config.connection_timeout_ms.has_value()){
        int connection_timeout=config.connection_timeout_ms.value()/1000; //获取值
        mysql_options(mysql_,MYSQL_OPT_CONNECT_TIMEOUT,&connection_timeout);
    }

    //如果“读超时”存在，就设置“读超时选项”
    if(config.read_timeout_ms.has_value()){
        int read_timeout=config.read_timeout_ms.value()/1000; //获取值
        mysql_options(mysql_,MYSQL_OPT_READ_TIMEOUT,&read_timeout);
    }

    //如果“写超时”存在，就设置“写超时选项”
    if(config.write_timeout_ms.has_value()){
        int write_timeout=config.write_timeout_ms.value()/1000; //获取值
        mysql_options(mysql_,MYSQL_OPT_WRITE_TIMEOUT,&write_timeout);
    }

    //如果“重连接”存在，就设置“重连接选项”
    if(config.auto_reconnect.has_value()){
        unsigned int mybool=config.auto_reconnect.value()?1:0; //获取值
        mysql_options(mysql_,MYSQL_OPT_RECONNECT,&mybool);
    }

    ...
}

MySQLConnection::MySQLConnection(const MySQLConfig& config){
	...

    if(!mysql_real_connect(mysql_,
                            config.host.c_str(),
                            config.username.c_str(),
                            config.password.c_str(),
                            config.database.c_str(),
                            config.port,
                            nullptr,
                            0)){
    	//连接失败
    	...
        
        //不重连的条件： 重连配置不存在 || 重连配置存在 但 设置为false ——> value_or(false) 
        bool should_retry = config.auto_reconnect.value_or(false);

        if (!should_retry) {
            LOG_ERROR("mysql_real_connect failed [{}]: {}", err_code, err_msg);
            throw MySQLException(err_code, err_msg);
        }
        
        //重连
        ConnectWithRetry(config);
    }
}

```