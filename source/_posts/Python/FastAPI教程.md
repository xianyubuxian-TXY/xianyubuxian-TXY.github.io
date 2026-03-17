---
title: FastAPI教程
date: 2026-03-04 22:26:02
tags:
	- 笔记
categories:
	- Python
	- FastAPI教程
---

# 一、FastAPI框架介绍
## 1.FastAPI简介
- **FastAPI** 是一个现代、快速（高性能）的 Python Web 框架，专门用于构建 API 服务。
- 它基于 **Starlette（高性能 ASGI）** 和 **Pydantic（数据校验）**，同时充分利用 Python 的类型注解。

## 2.FastAPI的主要特点
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260304223614249.png)

# 二、FastAPI基础教程
## 1.项目创建
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260304224546512.png)
- 使用**虚拟环境**：
	- 隔离项目运行环境，避免依赖冲突，保持全局环境的干净和稳定

### （1）项目结构介绍
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260304225227448.png)

### （2）main.py解析
```python
# 从 fastapi 模块导入 FastAPI 类
from fastapi import FastAPI

# 创建 FastAPI 应用实例
# 这是整个应用的核心对象，所有路由、中间件等都会注册到这个实例上
app = FastAPI()


# @app.get("/") 是一个装饰器，作用：
# 1. 定义 HTTP 请求方法为 GET
# 2. 定义路由路径为 "/" (根路径)
# 当用户访问 http://127.0.0.1:8000/ 时，会触发下面的函数
@app.get("/")   
async def root():
    """
    根路由处理函数
    
    - async: 声明为异步函数，支持异步操作（如数据库查询、网络请求等）
    - 返回字典会被 FastAPI 自动转换为 JSON 响应
    
    访问: GET http://127.0.0.1:8000/
    返回: {"message": "Hello World"}
    """
    return {"message": "Hello World"}


# @app.get("/hello/{name}") 定义了一个带路径参数的路由
# {name} 是路径参数，会从 URL 中提取
# 例如: /hello/张三 中的 "张三" 会被提取为 name 的值
@app.get("/hello/{name}")
async def say_hello(name: str):
    """
    带路径参数的路由处理函数
    
    参数:
        name (str): 路径参数，FastAPI 会自动：
                    1. 从 URL 中提取 {name} 的值
                    2. 根据类型注解 str 进行类型验证
                    3. 自动生成 API 文档
    
    访问示例: 
        GET http://127.0.0.1:8000/hello/张三
        GET http://127.0.0.1:8000/hello/FastAPI
    
    返回: 
        {"message": "Hello 张三"}
        {"message": "Hello FastAPI"}
    """
    # f-string 格式化字符串，将 name 变量嵌入到字符串中
    return {"message": f"Hello {name}"}
```

### （3）项目运行
在Terminal命令行的项目根目录中输入如下命令：
```bash
# 基础启动
uvicorn main:app

# 开发环境（热重载）
uvicorn main:app --reload

# 指定主机和端口
uvicorn main:app --host 0.0.0.0 --port 8000

# 生产环境（多 worker）
uvicorn main:app --workers 4

# 完整开发命令
uvicorn main:app --reload --host 127.0.0.1 --port 8080
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305154055784.png)

### （4）访问FastAPI交互式文档
```bash
http://127.0.0.1:8000/docs
```

## 2.路由
- 路由就是**URL地址**和**处理函数**之间的映射关系，它决定了当用户访问某个特定网址时，服务器应该执行哪段代码来返回结果。
- **FastAPI的路由定义基于Python的“装饰器模式”**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305155449017.png)
```python
# 访问 /hello  响应结果  msg: 你好 FastAPI
@app.get("/hello")
async def get_hello():
    return {"msg": "你好 FastAPI"}
```

## 3.FastAPI的参数类型
### （1）参数简介
- 参数就是客户端发送请求时，**附带的额外信息和指令**
- 参数的作用是让**同一个接口**能**根据不同的输入，返回不同的输出**，实现**动态交互**

### （2）参数分类
- **路径参数**：唯一标识/定位某个具体资源
- **查询参数**：对资源集合进行过滤、排序、分页等操作
- **请求体**：传递复杂/大量数据给服务器
```bash
┌─────────────────────────────────────────────────────┐
│  "定位某个资源"     →  路径参数    /users/{id}       │
│  "过滤/配置结果"    →  查询参数    ?page=1&sort=asc │
│  "提交复杂数据"     →  请求体      JSON body        │
└─────────────────────────────────────────────────────┘
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305160553546.png)

#### <1>路径参数
##### 1）路径参数 —— 基础使用
```python
async def get_book(id: int):
    return {"id": id, "title": f"这是第{id}本书"}
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305161333986.png)

##### 2）路径参数 —— 类型注解Path
- 在原生的Python中，允许对“路径参数”进行“类型注解”
- FastAPI 允许为参数声明**额外**的信息和校验（如：限定id范围等）
	- **"..."：表示“必填”，也可以使用“默认值”**
```python
from fastapi import FastAPI,Path
@app.get("/book/{id}")
async def get_book(id: int = Path(..., gt=0, lt=101, description="书籍的id，取值访问1-100")):
    return {"id": id, "title": f"这是第{id}本书"}

# 需求：查找书籍的作者，路径参数 name，长度范围 2-10
@app.get("/author/{name}")
async def get_name(name: str = Path(...,min_length=2,max_length=10)):
    return {"msg": f"这是{name}的信息"}
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305161841938.png)

#### <2>查询参数
##### 1）查询参数 —— 基础使用
- 声明的参数**不是路径参数**时，路径操作函数会把该参数**自动解析为查询参数**
```python
# 需求： 查询新闻 -> 分页，skip：跳过的记录数，limit：返回的记录数
@app.get("/news/news_list")
async def get_news_list(skip: int,limit: int = 10):
    return {"skip": skip, "limit": limit}
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305162902867.png)

##### 2）查询阐述 —— 类型注解Query
```python
from fastapi import FastAPI,Query
# 需求： 查询新闻 -> 分页，skip：跳过的记录数，limit：返回的记录数
@app.get("/news/news_list")
async def get_news_list(
    skip: int = Query(...,description="跳过的记录数",lt=100),
    limit: int = Query(10,description="返回的记录数") # ... 替换为具体值，即表示“默认值”
):
    return {"skip": skip, "limit": limit}
```

#### <3>请求体参数
##### 1）请求体参数 —— 基础使用
```python
from fastapi import  FastAPI
from pydantic import BaseModel

# 注册： 用户名和密码 - str
class User(BaseModel):
    username:str
    password:str

@app.post("/register")
async def register(user: User):
    return user
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305165406611.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305165446940.png)

##### 2）请求体参数 —— 类型注解Field
- 导入 pydantic 的 Field函数
```python
from fastapi import  FastAPI
from pydantic import BaseModel,Field

# 注册： 用户名和密码 - str
class User(BaseModel):
    username:str = Field(default="张三",min_length=2,max_length=10,description="用户名，长度要求2-10个字")
    password:str = Field(min_length=3,max_length=20)

@app.post("/register")
async def register(user: User):
    return user
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305170041061.png)

## 4.FastAPI的响应类型
### （1）响应类型简介
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305171218149.png)

### （2）响应类型设置方式
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305171542813.png)

#### <1>设置 HTML响应格式
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305171659718.png)

#### <2>设置 文件响应格式
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305171951373.png)

#### <3>自定义Json响应数据格式
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305172535389.png)

### （3）异常响应处理
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305172807422.png)

# 三、FastAPI进阶教程
## 1.中间件
- 使用中间件为每个请求前后添加统一的处理逻辑
- **使用场景**：多个接口使用统一的处理逻辑
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305173327862.png)

### （1）中间件简介
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305174026335.png)

### （2）中间件的使用
```python
@app.middleware("http")  # 装饰器：将此函数注册为 HTTP 中间件
async def middleware1(request, call_next):
    """
    HTTP 中间件函数
    
    参数:
        request (Request): 客户端发来的 HTTP 请求对象
                          包含: 请求头、请求体、URL、方法等
        
        call_next (Callable): 一个可调用对象，用于将请求传递给下一个处理环节
                             可能是下一个中间件，或最终的路由处理函数
    
    返回:
        Response: HTTP 响应对象
    """
    
    # ==================== 请求阶段 ====================
    # 在请求到达路由处理函数之前执行
    # 可以在这里：记录日志、验证身份、修改请求等
    print("中间件1 start")
    
    # ==================== 调用下一环节 ====================
    # await call_next(request) 做了什么：
    #   1. 将请求传递给下一个中间件（如果有）
    #   2. 或传递给实际的路由处理函数
    #   3. 等待处理完成，获取响应对象
    # 
    # 这是一个异步操作，会暂停当前中间件，等待后续处理完成
    response = await call_next(request)
    
    # ==================== 响应阶段 ====================
    # 在路由处理函数返回响应之后执行
    # 可以在这里：修改响应、添加响应头、记录耗时等
    print("中间件1 end")
    
    # 返回响应给上一层（或直接返回给客户端）
    return response

```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305173717350.png)

### （3）中间件的执行流程
- 请求时：**自下而上**
- 响应时：**自上而下**
```python
from fastapi import FastAPI

app = FastAPI()

@app.middleware("http")
async def middleware1(request, call_next):
    print("中间件1 start")
    response = await call_next(request)
    print("中间件1 end")
    return response

@app.middleware("http")
async def middleware2(request, call_next):
    print("中间件2 start")
    response = await call_next(request)
    print("中间件2 end")
    return response

@app.get("/")
async def root():
    return {"message": "Hello World"}
```
访问http://127.0.0.1:8000 后，发起"http"请求，在bash输出如下内容：
```bash
中间件2 start
中间件1 start
中间件1 end
中间件2 end
INFO:     127.0.0.1:51561 - "GET / HTTP/1.1" 200 OK
```
#### 洋葱模型图解
```python
        请求进入
            │
            ▼
    ┌───────────────────────────────┐
    │  middleware2 （后注册 = 外层）  │
    │  ┌─────────────────────────┐  │
    │  │  middleware1 （先注册）   │  │
    │  │  ┌───────────────────┐  │  │
    │  │  │                   │  │  │
① ─▶│  │  │                   │  │  │
    │  │  │                   │  │  │
  ②─│─▶│  │    @app.get("/")  │  │  │
    │  │  │    路由处理函数     │  │  │
    │  │  │                   │◀─│─③│
    │  │  │                   │  │  │
    │  │  └───────────────────┘◀④│  │
    │  └─────────────────────────┘  │
    └───────────────────────────────┘
            │
            ▼
        响应返回

中间件2 start    ← 第1个执行（后注册的先执行）
中间件1 start    ← 第2个执行
中间件1 end      ← 第3个执行（先进后出）
中间件2 end      ← 第4个执行
```

## 2.依赖注入
- 使用**依赖注入系统**来“共享通用逻辑”，减少代码重复
- **与中间件的区别**：
	- **中间件**：所有接口都会自动执行“中间件”中的代码，适用于**“所有接口”的通用逻辑**
	- **依赖注入**：哪个接口需要“通用代码”，就像封装好的“通用代码”注入，只有“注入的接口”会执行“注入的代码”

### （1）依赖注入简介
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305194122960.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305194139159.png)

### （2）依赖注入的使用
```python
from fastapi import FastAPI,Query,Depends
# 把 分页参数逻辑公用： 新闻列表、用户列表
# 创建依赖项
async def common_parameters(
    skip: int = Query(0,ge=0),
    limit: int = Query(10,le=60)
):
    return {"skip": skip, "limit": limit}

# 进行依赖注入
@app.get("/news/news_list")
async def get_news_list(commons= Depends(common_parameters)):
    return commons

@app.get("/users/users_list")
async def get_users_list(commons= Depends(common_parameters)):
    return commons
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260305195350624.png)


## 3.ORM
### （1）ORM简介与安装
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306093416667.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306093436906.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306093454514.png)

### （2）SQLAlchemy ORM安装
```bash
pip install sqlalchemy[asyncio] aiomysql
```

### （3）ORM 建表
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306094046346.png)

**注**：需要提前建立好相应的数据库
```bash
create database fastapi_first;
```
```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Query, Depends
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import DateTime, func, Integer, String, Float, URL
from datetime import datetime

# ============================================================
# 1. 创建数据库异步引擎
# ============================================================
ASYNC_DATABASE_URL = URL.create(
    drivername="mysql+aiomysql",
    username="root",
    password="@txylj2864",
    host="localhost",
    port=3306,
    database="fastapi_first",
    query={"charset": "utf8"}
)

engine = create_async_engine(
    ASYNC_DATABASE_URL,
    echo=True,
    pool_size=10,
    max_overflow=20
)


# ============================================================
# 2. 定义模型类
# ============================================================
class Base(DeclarativeBase):
    create_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), default=func.now(), comment="创建时间"
    )
    update_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), default=func.now(), onupdate=func.now(), comment="修改时间"
    )


class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, comment="书籍id")
    bookname: Mapped[str] = mapped_column(String(255), comment="书名")
    author: Mapped[str] = mapped_column(String(255), comment="作者")
    price: Mapped[float] = mapped_column(Float, comment="书籍价格")
    publisher: Mapped[str] = mapped_column(String(255), comment="出版社")


# ============================================================
# 3. 定义 lifespan（启动建表 + 关闭释放连接）
# ============================================================
@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    应用生命周期管理
    """
    # ========== 启动时执行 ==========
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    print("✅ 数据库表创建完成")

    yield  # 应用运行中...

    # ========== 关闭时执行 ==========
    await engine.dispose()
    print("🔌 数据库连接池已关闭")


# ============================================================
# 4. 创建 FastAPI 实例（传入 lifespan）
# ============================================================
app = FastAPI(lifespan=lifespan)


# ============================================================
# 5. 路由示例
# ============================================================
@app.get("/")
async def root():
    return {"message": "Hello World"}
```

#### <1>创建异步数据库引擎
```python
from sqlalchemy.ext.asyncio import create_async_engine
# 1.创建数据库异步引擎
ASYNC_DATABASE_URL = URL.create(
    drivername="mysql+aiomysql",
    username="root",
    password="@txylj2864",  # 原始密码，自动处理特殊字符@
    host="localhost",
    port=3306,
    database="fastapi_first",
    query={"charset": "utf8"}
)

engine = create_async_engine(
    ASYNC_DATABASE_URL,
    echo=True,          # 【可选】输出SQL日志
    pool_size=10,       # 设置连接池活跃的连接数
    max_overflow=20     # 允许额外的连接数
)
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306094154418.png)

#### <2>定义模型类
**基本语法**：
```python
属性名: Mapped[Python类型] = mapped_column(SQL类型, 约束参数...)
```
```python
from sqlalchemy.orm import DeclarativeBase,Mapped,mapped_column
from sqlalchemy import DateTime, func, Integer, String, Float
from datetime import datetime
# 2.定义模型类： 基类 + 表对应的模型类
# 基类：创建时间、更新时间； 书籍表：id、书名、作者、价格、出版社
class Base(DeclarativeBase):
    """
    DeclarativeBase: SQLAlchemy 2.0 的声明式基类
    所有模型类都要继承它
    """
    
    create_time: Mapped[datetime] = mapped_column(
        DateTime,                    # SQL 列类型：DATETIME
        insert_default=func.now(),   # INSERT 时的数据库端默认值（由数据库生成）
        default=func.now(),          # Python 端默认值（ORM 层面）
        comment="创建时间"            # 数据库列注释
    )
    
    update_time: Mapped[datetime] = mapped_column(
        DateTime,
        insert_default=func.now(),
        default=func.now(),
        onupdate=func.now(),         # UPDATE 时自动更新为当前时间
        comment="修改时间"
    )

class Book(Base):
    __tablename__ = "book"           # 指定数据库表名
    
    id: Mapped[int] = mapped_column(
        Integer,                     # SQL 列类型：INTEGER
        primary_key=True,            # 主键
        comment="书籍id"
    )
    
    bookname: Mapped[str] = mapped_column(
        String(255),                 # SQL 列类型：VARCHAR(255)
        comment="书名"
    )
    
    author: Mapped[str] = mapped_column(
        String(255),
        comment="作者"
    )
    
    price: Mapped[float] = mapped_column(
        Float,                       # SQL 列类型：FLOAT
        comment="书籍价格"
    )
    
    publisher: Mapped[str] = mapped_column(
        String(255),
        comment="出版社"
    )
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306095051458.png)

#### <3>创建数据库表
- **Lifespan语法**：
```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    @asynccontextmanager  → 异步上下文管理器装饰器
    app: FastAPI          → 接收 FastAPI 实例（可访问 app.state）
    """
    
    # ========== STARTUP ==========
    # yield 之前的代码在应用启动时执行
    print("启动")
    
    yield  # ← 暂停点，应用在此运行
    
    # ========== SHUTDOWN ==========
    # yield 之后的代码在应用关闭时执行
    print("关闭")

# 关键：将 lifespan 传入 FastAPI
app = FastAPI(lifespan=lifespan)
```
```python
# ============================================================
# 3. 定义 lifespan（启动建表 + 关闭释放连接）
# ============================================================
@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    应用生命周期管理
    """
    # ========== 启动时执行 ==========
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    print("✅ 数据库表创建完成")
    
    yield  # ← 在这里【暂停】，控制权交给 FastAPI
           #   FastAPI 开始接收请求
           #   lifespan 函数在这里【等待】应用关闭信号
    
    # ========== 关闭时执行 ==========
    # 只有收到关闭信号（Ctrl+C）后，才会执行到这里
    await engine.dispose()
    print("🔌 数据库连接池已关闭")



# ============================================================
# 4. 创建 FastAPI 实例（传入 lifespan）
# ============================================================
app = FastAPI(lifespan=lifespan)


# ============================================================
# 5. 路由示例
# ============================================================
@app.get("/")
async def root():
    return {"message": "Hello World"}
```
**📊 执行流程**
```python
uvicorn main:app --reload
         │
         ▼
┌─────────────────────────────────┐
│  lifespan() 开始执行             │
│         │                       │
│         ▼                       │
│  engine.begin() 开启事务         │
│         │                       │
│         ▼                       │
│  Base.metadata.create_all       │
│  → CREATE TABLE IF NOT EXISTS   │
│         │                       │
│         ▼                       │
│  print("✅ 表创建完成")           │
│         │                       │
│         ▼                       │
│       yield ←───────────────────│── 应用开始接收请求
│         │                       │
│         │                       │
└────────┬────────────────────────┘
         │
┌────────┴────────────────────────────────────────┐
│                                                 │
│      此时 FastAPI 应用在【外部】运行               │
│                                                 │
│      uvicorn 事件循环处理 HTTP 请求               │
│           /book/books  →  get_books_list()      │
│           /            →  root()                │
│                                                 │
│      lifespan 函数一直卡在 yield 这里等着         │
│                                                 │
└────────┬────────────────────────────────────────┘
         │
   Ctrl+C 停止服务器
         │
         ▼
┌─────────────────────────────────┐
│  yield 之后继续执行              │
│         │                       │
│         ▼                       │
│  engine.dispose() 关闭连接池     │
│         │                       │
│         ▼                       │
│  print("🔌 连接已关闭")          │
│         │                       │
│         ▼                       │
│      程序结束                    │
└─────────────────────────────────┘

```

## 4.路由匹配中使用 ORM
- **核心**：创建“依赖项”，使用 Depends 注入到处理函数
```python
from sqlalchemy.ext.asyncio import async_sessionmaker, AsyncSession

# 1.创建异步会话工厂
async_sessionmaker_local = async_sessionmaker(
    bind=engine,            # 绑定数据库引擎（之前创建的 create_async_engine）
    class_=AsyncSession,    # 指定会话类型为异步会话
    expire_on_commit=False  # 提交后对象不过期，避免访问属性时重新查询数据库
)
"""
async_sessionmaker: 异步会话工厂（类似于工厂模式）
    - 调用 async_sessionmaker_local() 就能创建一个新的 AsyncSession
    - 每个请求应该使用独立的 session
"""

# 2.创建依赖项
async def get_database():
    """
    async def: 异步生成器函数（因为有 yield）
    用于 FastAPI 的 Depends() 依赖注入
    """
    
    async with async_sessionmaker_local() as session:
        """
        async_sessionmaker_local()  → 创建一个新的 AsyncSession 实例
        async with ... as session   → 异步上下文管理器，自动管理会话生命周期
        """
        
        try:
            yield session  # 生成器语法
            """
            yield session:
                - 将 session 提供给路由函数使用
                - 函数在这里暂停，等待路由函数执行完毕
                - 路由函数通过 Depends(get_database) 获取这个 session
            """
            
            await session.commit()
            """
            路由函数正常执行完毕后：
                - 提交事务，保存所有更改到数据库
            """
            
        except Exception:
            await session.rollback()
            """
            如果路由函数抛出异常：
                - 回滚事务，撤销所有未提交的更改
            """
            raise  # 重新抛出异常，让 FastAPI 处理
            
        finally:
            await session.close()
            """
            无论成功或失败：
                - 关闭会话，释放数据库连接回连接池
            """

# 3.创建 FastAPI 实例（传入 lifespan）
app = FastAPI(lifespan=lifespan)

# 依赖注入
@app.get("/book/books")
async def get_books_list(db: AsyncSession = Depends(get_database)):
    # 查询
    result = await db.execute(select(Book))
    book = result.scalars().all()
    return book
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306103214180.png)
**🔄 执行流程图**
```python
路由函数调用 Depends(get_database)
                │
                ▼
┌───────────────────────────────────────────┐
│  get_database() 开始执行                    │
│                │                           │
│                ▼                           │
│  async_sessionmaker_local() 创建 session   │
│                │                           │
│                ▼                           │
│           try 开始                          │
│                │                           │
│                ▼                           │
│          yield session ──────────────────────▶ 路由函数获得 session
│                │                           │         │
│          （暂停等待）                        │    执行数据库操作
│                │                           │         │
│                │◀──────────────────────────────────────┘
│                │                           │
│         路由执行完毕？                       │
│           ／    ＼                          │
│         成功     异常                        │
│          │        │                         │
│          ▼        ▼                         │
│     commit()   rollback()                   │
│          │        │                         │
│          └───┬────┘                         │
│              ▼                              │
│         finally: close()                    │
└───────────────────────────────────────────┘
```

## 5.ORM数据操作
### （1）查询操作
#### <1>基础查询
```python
@app.get("/book/books")
async def get_books_list(db: AsyncSession = Depends(get_database)):
    # 查询
    # result = await db.execute(select(Book))   # 查询 -> 返回一个ORM对象
    # book = result.scalars().all() # 获取所有数据
    # book = result.scalars().first() # 获取第一条数据

    book = await db.get(Book,3) # 获取单条数据 —> 根据主键
    return book
```
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306112009662.png)

#### <2>条件查询（where）
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306113033661.png)
```python
# =========================== 1.条件查询 ===========================
# 需求： 路径参数（书籍id）
@app.get("/book/get_book/{book_id}")
async def get_book_lsit(book_id: int, db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.id == book_id))
    book = result.scalar_one_or_none()  # 主键查询，唯一 ——> 存在，则返回一条数据； 不存在，则返回None
    return book

# 需求： 条件（价格 >= 150）
@app.get("/book/search_book")
async def get_book_search(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.price >= 150))
    book = result.scalars().all()  # 非主键查询 ——> 返回所有满足的数据
    return book

# =========================== 2.模糊查询 + 逻辑运算符 + in_() ===========================
# 需求： 作者以“曹”开头 (% _)  & pricce>150
@app.get("/book/search_book")
async def get_search_book(db: AsyncSession = Depends(get_database)):
    # like() 模糊查询 : % 任意个字符； _ 一个单个字符
    # @ | ~ : 逻辑运算符
    # in_() : 包含

    result = await db.execute(select(Book).where( (Book.author.like("曹%")) & (Book.price > 150) ))
    # result = await db.execute(select(Book).where(Book.author.like("曹_")))
    books = result.scalars().all()

    # 需求： 书籍id列表，数据库里面的id如果在书籍id列表里面，就返回
    id_list = [1,2,3]
    result2 = await db.execute(select(Book).where(Book.id.in_(id_list)))
    books2 = result2.scalars().all()

    return books,books2
```

#### <3>聚合查询
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306115458559.png)
```python
@app.get("/book/count")
async def get_count(db: AsyncSession = Depends(get_database)):
    # 聚合查询 select(func.聚合方法名（模型类.属性）)
    # result = await db.execute(select(func.max(Book.price)))   # 最大值
    # result = await db.execute(select(func.min(Book.price)))   # 最小值
    # result = await db.execute(select(func.sum(Book.price)))   # 求和
    # result = await db.execute(select(func.avg(Book.price)))   # 平均值
    result = await db.execute(select(func.count(Book.id)))      # 数量统计
    num = result.scalar()   # 用来提取一个数值 ——> 标量值
    return num
```

#### <4>分页查询
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306120233151.png)
```python
@app.get("/book/get_book_list")
async def get_book_list(
    page: int=1,
    page_size: int=2,
    db: AsyncSession = Depends(get_database)
):
    # skip = (页码 -1) * 每页数量
    skip = (page-1) * page_size
    # offset: 跳过的记录数   limit: 每页的记录数
    stmt = select(Book).offset(skip).limit(page_size)
    result = await db.execute(stmt)
    books = result.scalars().all()
    return books
```

#### <5>排序查询
```python
# 获取相关新闻
async def get_related_news(db: AsyncSession,news_id: int,category_id: int,limit: int = 5):
    # order_by 排序 -> 浏览量和发布时间
    stmt=select(News).where(
        News.category_id == category_id,
        News.id != news_id
    ).order_by(
        News.views.desc(),  # 默认是升序， desc 表示降序
        News.publish_time.desc()
    ).limit(limit)
    result=await db.execute(stmt)
    return result.scalars().all()
```

### （2）增加操作
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306120944023.png)
```python
# 需求： 用户输入图书信息（id、书名、作者、价格、出版社） —> 新增
# 用户输入 —> 参数 —> 请求体
from pydantic import BaseModel
# BaseModel 用于请求体参数： 用于自动校验请求数据的类型和格式
class BookBase(BaseModel):
    id: int
    bookname: str
    author: str
    price: float
    publisher: str

@ app.post("/book/add_book")
async def add_book(book: BookBase, db: AsyncSession = Depends(get_database)):
    book_obj = Book(**book.__dict__)   # 将书籍信息类 转为 ORM类
    db.add(book_obj)
    await db.commit()
    return book_obj
```

### （3）更新操作
```python
# 需求：修改图书信息（先查再改）
# 设计思路： 路径参数书籍id（查找），请求体参数（书籍新信息：书名、作者、价格、出版社）
from pydantic import BaseModel

class BookUpdate(BaseModel):
    bookname: str
    author: str
    price: float
    publisher: str

@app.put("/book/update_book/{book_id}")
async def update_book(book_id: int,data: BookUpdate, db: AsyncSession = Depends(get_database)):
    # 1.查找图书
    db_book = await db.get(Book,book_id)
    if db_book is None:     # 如果未找到，则抛出异常
        raise HTTPException(status_code=404,detail="要修改的书籍不存在")

    # 2.如果找到：重新赋值
    db_book.bookname=data.bookname
    db_book.author=data.author
    db_book.price=data.price
    db_book.publisher=data.publisher

    # 3.提交到数据库
    await db.commit()
    return db_book

# 浏览数+1
async def increase_news_views(db: AsyncSession,news_id: int):
    stmt = update(News).where(News.id==news_id).values(views=News.views+1)
    await db.execute(stmt)
    await db.commit()
```
```python
# 更新用户信息的模型类
class UpdateUserRequest(BaseModel):
    nickname: str=None
    avatar: str=None
    gender: str=None
    bio: str=None
    phone: str=None


# 更新用户信息
async def update_user_info(db: AsyncSession, username: str, user_data: UpdateUserRequest):
    """
    异步更新用户信息
    :param db: 数据库会话
    :param username: 要更新的用户名
    :param user_data: Pydantic 模型，包含要更新的字段
    :return: 更新后的用户对象
    """
    
    # update().where(User.username == username).values.(字段=值，字段=值)
    # user_data 是一个Pydantic类型，得到字典 -> **解包
    
    # 构建 UPDATE 语句
    # ============================================================
    # model_dump() 是 Pydantic v2 的方法（v1 中叫 .dict()）
    # 功能：将 Pydantic 模型实例转换为 Python 字典
    # 
    # 示例：
    #   user_data = UpdateUserRequest(bio="新简介", avatar=None)
    #   user_data.model_dump()                        → {"bio": "新简介", "avatar": None, "nickname": None}
    #   user_data.model_dump(exclude_unset=True)      → {"bio": "新简介", "avatar": None}  # 排除未设置的
    #   user_data.model_dump(exclude_none=True)       → {"bio": "新简介"}                  # 排除None值
    #   user_data.model_dump(exclude_unset=True, exclude_none=True) → {"bio": "新简介"}    # 两者结合
    #
    # 参数说明：
    #   exclude_unset=True  → 排除用户没有传的字段（保留数据库原值）
    #   exclude_none=True   → 排除值为 None 的字段（避免覆盖为空）
    # 
    # **解包 → 将字典展开为 key=value 形式传入 values()
    #   例: **{"bio": "新简介"} → bio="新简介"
    # ============================================================
    stmt = update(User).where(User.username == username).values(
        **user_data.model_dump(
            exclude_unset=True,
            exclude_none=True
        )
    )
    
    # 执行 UPDATE 语句
    result = await db.execute(stmt)
    
    # 提交事务，使更改持久化到数据库
    await db.commit()
    
    # 检查更新 - rowcount 表示受影响的行数
    # 如果为 0，说明没有匹配的用户
    if result.rowcount == 0:
        raise HTTPException(status_code=404, detail="用户不存在")
    
    # 重新查询获取更新后的用户数据
    # （因为 UPDATE 语句不会返回更新后的对象）
    updated_user = await get_user_by_username(db, username)
    
    return updated_user

```

![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306161136077.png)

### （4）删除操作
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306161451885.png)


## 6.注册"全局异常处理器""
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260307173809301.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260307173824993.png)
```python
# exception.py
import traceback
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse
from sqlalchemy.exc import IntegrityError, SQLAlchemyError
from starlette import status

# 开发模式:返回详细错误信息 # 生产模式:返回简化错误信息
DEBUG_MODE = True  # 教学项目保持开启

async def http_exception_handler(request: Request, exc: HTTPException):
    """
    处理 HTTPException 异常
    """
    # HTTPException 通常是业务逻辑主动抛出的,data 保持 None
    return JSONResponse(
        status_code=exc.status_code,
        content={"code": exc.status_code, "message": exc.detail, "data": None}
    )

async def integrity_error_handler(request: Request, exc: IntegrityError):
    """
    处理数据库完整性约束错误
    """
    error_msg = str(exc.orig)
    # 判断具体的约束错误类型
    if "username_UNIQUE" in error_msg or "Duplicate entry" in error_msg:
        detail = "用户名已存在"
    elif "FOREIGN KEY" in error_msg:
        detail = "关联数据不存在"
    else:
        detail = "数据约束冲突,请检查输入"
    # 开发模式下返回详细错误信息
    error_data = None
    if DEBUG_MODE:
        error_data = {
            "error_type": "IntegrityError",
            "error_detail": error_msg,
            "path": str(request.url)
        }
    return JSONResponse(
        status_code=status.HTTP_400_BAD_REQUEST,
        content={"code": 400, "message": detail, "data": error_data}
    )

async def sqlalchemy_error_handler(request: Request, exc: SQLAlchemyError):
    """
    处理 SQLAlchemy 数据库错误
    """
    # 开发模式下返回详细错误信息
    error_data = None
    if DEBUG_MODE:
        error_data = {
            "error_type": type(exc).__name__,
            "error_detail": str(exc),
            "traceback": traceback.format_exc(),
            "path": str(request.url)
        }
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "code": 500,
            "message": "数据库操作失败,请稍后重试",
            "data": error_data
        }
    )

async def general_exception_handler(request: Request, exc: Exception):
    """
    处理所有未捕获的异常
    """
    # 开发模式下返回详细错误信息
    error_data = None
    if DEBUG_MODE:
        error_data = {
            "error_type": type(exc).__name__,
            "error_detail": str(exc),
            # 格式化异常信息为字符串,方便日志记录和调试
            "traceback": traceback.format_exc(),
            "path": str(request.url)
        }
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={
            "code": 500,
            "message": "服务器内部错误",
            "data": error_data
        }
    )
```
```python
# exception_handlers.py
from fastapi import HTTPException
from sqlalchemy.exc import IntegrityError,SQLAlchemyError

from utils.exception import http_exception_handler, integrity_error_handler, sqlalchemy_error_handler, \
    general_exception_handler


def register_exception_handlers(app):
    """
    注册全局异常处理：子类在前，父类在后；具体在前，抽象在后
    """
    app.add_exception_handler(HTTPException, http_exception_handler)    # 业务
    app.add_exception_handler(IntegrityError,integrity_error_handler)   # 数据完整性约束
    app.add_exception_handler(SQLAlchemyError,sqlalchemy_error_handler) # 数据库
    app.add_exception_handler(Exception,general_exception_handler)      # 兜底
```
```python
# main.py
from fastapi import FastAPI
from routers import news, users
from fastapi.middleware.cors import CORSMiddleware

from utils.exception_handlers import register_exception_handlers

app = FastAPI()

# 注册异常处理器
register_exception_handlers(app)
```

# 四、掘金头条项目实战
## 0.项目结构
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306211354120.png)

## 1.模块化路由
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306212050905.png)
```python
# routers/news.py

# ============================================================
# 导入模块
# ============================================================
from fastapi import APIRouter
# APIRouter: FastAPI 提供的路由器类
# 作用: 用于创建模块化的路由，可以把不同功能的接口分开管理
# 类似于 Flask 的 Blueprint（蓝图）

# ============================================================
# 1. 创建 API 路由器实例
# ============================================================
router = APIRouter(prefix="/api/news", tags=["news"])
# 
# 参数说明:
# ┌─────────────────────────────────────────────────────────────┐
# │ prefix="/api/news"                                          │
# │   路由前缀，这个路由器下的所有接口都会自动加上这个前缀         │
# │   例如: 定义 "/categories" → 实际访问地址是 "/api/news/categories" │
# ├─────────────────────────────────────────────────────────────┤
# │ tags=["news"]                                               │
# │   API 标签/分组名称                                          │
# │   作用: 在 Swagger 文档（/docs）中，这个路由器的所有接口       │
# │        会被归类到 "news" 分组下，方便查看和管理               │
# └─────────────────────────────────────────────────────────────┘

# ============================================================
# 2. 定义路由/接口
# ============================================================
@router.get("/categories")
# @router.get() 是一个装饰器
# 
# 参数说明:
# ┌─────────────────────────────────────────────────────────────┐
# │ .get        → HTTP 请求方法为 GET（用于获取数据）            │
# │ "/categories" → 路由路径                                    │
# │                                                             │
# │ 完整访问地址 = prefix + path                                 │
# │             = "/api/news" + "/categories"                   │
# │             = "/api/news/categories"                        │
# └─────────────────────────────────────────────────────────────┘
#
# 其他常用 HTTP 方法:
#   @router.post()   - 创建数据 (Create)
#   @router.put()    - 更新数据 (Update) 
#   @router.delete() - 删除数据 (Delete)
#   @router.patch()  - 部分更新数据

async def get_news_categories():
    # async def: 定义异步函数
    # ┌─────────────────────────────────────────────────────────┐
    # │ async: 表示这是一个异步函数（协程）                       │
    # │        FastAPI 推荐使用 async，可以处理高并发请求         │
    # │        如果函数内有 I/O 操作（数据库、网络请求等）         │
    # │        使用 async 可以提高性能                           │
    # ├─────────────────────────────────────────────────────────┤
    # │ 函数名: get_news_categories                              │
    # │ 命名建议: 动词 + 名词，清晰表达接口功能                    │
    # └─────────────────────────────────────────────────────────┘
    
    return {"message": "获取分类成功"}
    # 返回值:
    # - FastAPI 会自动将 Python 字典转换为 JSON 格式响应
    # - 自动设置 Content-Type: application/json
    # - 默认状态码: 200 OK
    #
    # 返回结果示例:
    # {
    #     "message": "获取分类成功"
    # }

```
```python
# main.py

from fastapi import FastAPI
from routers import news # 导包
app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}


# 挂载路由/注册路由
app.include_router(news.router)
```

## 2.数据库与ORM配置
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306212730657.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306212745600.png)
- 通过sql文件建表，建表部分就只需要“创建异步引擎”，用于“操作数据”即可
```python
# config/db_config.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from sqlalchemy import URL


# ============================================================
# 创建数据库异步引擎
# ============================================================
ASYNC_DATABASE_URL = URL.create(
    drivername="mysql+aiomysql",
    username="root",
    password="@txylj2864",
    host="localhost",
    port=3306,
    database="news_app",
    query={"charset": "utf8"}
)

engine = create_async_engine(
    ASYNC_DATABASE_URL,
    echo=True,
    pool_size=10,
    max_overflow=20
)

# ============================================================
# 创建异步会话工厂
# ============================================================
async_sessionmaker_local = async_sessionmaker(
    bind=engine,            # 绑定数据库引擎（之前创建的 create_async_engine）
    class_=AsyncSession,    # 指定会话类型为异步会话
    expire_on_commit=False  # 提交后对象不过期，避免访问属性时重新查询数据库
)

# ============================================================
# 创建依赖项：用于获取数据库会话
# ============================================================
async def get_database():
    async with async_sessionmaker_local() as session:
        try:
            yield session  # 生成器语法
            await session.commit()

        except Exception:
            await session.rollback()
            raise

        finally:
            await session.close()
```

## 3.业务模块
### （1）新闻模块
**新闻模块核心功能**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306213816441.png)

**接口实现流程**
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306214015825.png)

#### <1>获取新闻分类
```python
# 接口实现流程
# 1.模块化路由 -> API 接口规范文档
# 2.定义模型类 -> 数据库表（数据库设计文档）
# 3.在 crud 文件夹里面创建文件，封装操作数据库的方法
# 4.在路由处理函数里面调用crud封装好的方法，响应结果
```
```python
# config/db_config.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

from sqlalchemy import URL


# ============================================================
# 创建数据库异步引擎
# ============================================================
ASYNC_DATABASE_URL = URL.create(
    drivername="mysql+aiomysql",
    username="root",
    password="@txylj2864",
    host="localhost",
    port=3306,
    database="news_app",
    query={"charset": "utf8"}
)

engine = create_async_engine(
    ASYNC_DATABASE_URL,
    echo=True,
    pool_size=10,
    max_overflow=20
)

# ============================================================
# 创建异步会话工厂
# ============================================================
async_sessionmaker_local = async_sessionmaker(
    bind=engine,            # 绑定数据库引擎（之前创建的 create_async_engine）
    class_=AsyncSession,    # 指定会话类型为异步会话
    expire_on_commit=False  # 提交后对象不过期，避免访问属性时重新查询数据库
)

# ============================================================
# 创建依赖项：用于获取数据库会话
# ============================================================
async def get_db():
    async with async_sessionmaker_local() as session:
        try:
            yield session  # 生成器语法
            await session.commit()

        except Exception:
            await session.rollback()
            raise

        finally:
            await session.close()
```
```python
# models/news.py
from datetime import datetime

from sqlalchemy import Integer, String, DateTime
from sqlalchemy.orm import DeclarativeBase,Mapped,mapped_column

class Base(DeclarativeBase):
    created_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.now,
        comment="创建时间"
    )

    updated_at: Mapped[datetime] = mapped_column(
        DateTime,
        default=datetime.now,
        onupdate=datetime.now,
        comment="更新时间"
    )

# “新闻分类”模型类
class NewsCategory(Base):
    __tablename__ = "news_category"
    id: Mapped[int] = mapped_column(
        Integer,
        primary_key=True,
        autoincrement=True,
        comment="新闻分类id"
    )
    name: Mapped[str] = mapped_column(
        String(32),
        unique=True,
        nullable=False,
        comment="新闻分类名称"
    )
    sort_order: Mapped[int] = mapped_column(
        Integer,
        default=0,
        nullable=False,
        comment="排序"
    )

    def __repr__(self):
        return f"<NewsCategory(id={self.id}, name={self.name}, sort_order={self.sort_order})>"
```
```python
# routers/news.py
from fastapi import APIRouter,Depends
from sqlalchemy.ext.asyncio import AsyncSession

from config.db_config import get_db
from crud import news

# 创建 APIRouter 实例
# prefiex: 路由前缀（API 接口规范文档）
# tags 分组 标签
router = APIRouter(prefix="/api/news", tags=["news"])



# 实现路由
@router.get("/categories")
async def get_news_categories(db :AsyncSession = Depends(get_db),skip: int = 0, limit: int = 100):
    # 获取数据库里面新闻分类数据：先定义模型类，在封装查询数据的方法
    categories = await news.get_news_category(db,skip, limit)
    return {
        "code": 200,
        "message": "获取新闻分类成功",
        "data": categories
    }
```
```python
# main.py
from fastapi import FastAPI
from routers import news
from fastapi.middleware.cors import CORSMiddleware
app = FastAPI()

# 添加“跨域中间件”：解决跨域问题
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],    # 允许的源，开发阶段允许所有源，生产环境需要指定源
    allow_credentials=True, # 允许携带cookie
    allow_methods=["*"],    # 允许的请求方法
    allow_headers=["*"],    # 允许的请求头
)

@app.get("/")
async def root():
    return {"message": "Hello World"}


# 挂载路由/注册路由
app.include_router(news.router)

```

## 补充1：解决跨域问题
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306223734150.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306223751643.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260306223810019.png)
```python
from fastapi import FastAPI
from routers import news, users
from fastapi.middleware.cors import CORSMiddleware

from utils.exception_handlers import register_exception_handlers

app = FastAPI()

# 添加“跨域中间件”：解决跨域问题
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],    # 允许的源，开发阶段允许所有源，生产环境需要指定源
    allow_credentials=True, # 允许携带cookie
    allow_methods=["*"],    # 允许的请求方法
    allow_headers=["*"],    # 允许的请求头
)

@app.get("/")
async def root():
    return {"message": "Hello World"}


# 挂载路由/注册路由
app.include_router(news.router)
app.include_router(users.router)
```


## 补充2：密码加密
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260307155140115.png)
```python
pip install "passlib[bcrypt]==1.7.4" "bcrypt==4.0.1"
```
```python
from passlib.context import CryptContext

# 创建密码上下文
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# 密码加密
def get_hash_password(password:str):
    return pwd_context.hash(password)

# 验证密码: verify 返回值是布尔值
def verfy_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)
```

## 补充3：封装统一响应接口
```python
from typing import TypeVar, Generic, Optional
from pydantic import BaseModel, ConfigDict

# 定义泛型类型变量 T，用于表示 data 字段的类型
# T 可以是任意类型：UserInfoResponse、List[User]、str 等
T = TypeVar("T")


class Result(BaseModel, Generic[T]):
    """
    统一 API 响应结构

    继承说明：
    - BaseModel: Pydantic 基类，提供数据验证和序列化
    - Generic[T]: Python 泛型，让 data 字段类型可变

    使用示例：
    - Result[User]      -> data 是 User 类型
    - Result[List[str]] -> data 是字符串列表
    - Result[None]      -> data 为空
    """

    code: int = 200  # 响应状态码，默认 200
    message: str = "success"  # 响应消息，默认 "success"
    data: Optional[T] = None  # 响应数据，类型由泛型 T 决定，可为空

    # Pydantic v2 配置
    model_config = ConfigDict(
        from_attributes=True  # 允许从 ORM 对象属性中取值（如 SQLAlchemy 模型）
    )

    @classmethod
    def ok(cls, data: T = None, message: str = "success"):
        """
        成功响应的工厂方法

        Args:
            data: 响应数据，类型由调用时的泛型决定
            message: 成功消息，默认 "success"

        Returns:
            Result 实例，code=200

        示例：
            Result.ok(data=user_info)
            Result.ok(data=user_list, message="查询成功")
        """
        return cls(code=200, message=message, data=data)

    @classmethod
    def fail(cls, message: str = "error", code: int = 400):
        """
        失败响应的工厂方法

        Args:
            message: 错误消息
            code: 错误码，默认 400

        Returns:
            Result 实例，data=None

        示例：
            Result.fail(message="用户名或密码错误", code=401)
            Result.fail(message="参数错误")
        """
        return cls(code=code, message=message, data=None)
```
```python
from typing import Optional

from pydantic import BaseModel, ConfigDict,Field

# user_info 对应的类： 基础类 + Info类 （id、用户名）
class UserInfoBase(BaseModel):
    """
    用户信息基础字段
    这些字段是可选的，用于用户资料的扩展信息
    """
    nickname: Optional[str] = Field(None, max_length=50, description="昵称")
    avatar: Optional[str] = Field(None, max_length=255, description="头像URL")
    gender: Optional[str] = Field(None, max_length=10, description="性别")
    bio: Optional[str] = Field(None, max_length=500, description="个人简介")


class UserInfoResponse(UserInfoBase):
    """
    用户信息响应模型
    继承 UserInfoBase，添加必要的用户标识字段
    """
    id: int
    username: str

    model_config = ConfigDict(
        populate_by_name=True,       # 支持字段名和别名两种方式赋值
        from_attributes=True         # 允许从 ORM 对象属性中取值（如 SQLAlchemy 模型）
    )


# 返回值中，data 数据类型
class UserAuthResponse(BaseModel):
    token: str
    user_info: UserInfoResponse = Field(...,alias="userInfo")

    # 模型配置
    model_config = ConfigDict(
        populate_by_name=True,  # alias / 字段名兼容
        from_attributes= True   # 允许从 ORM 对象属性中取值
    )
```

## 补充4：生成Token
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260307161219602.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260307161435127.png)
```python
# 生成 Token
async def create_token(db:AsyncSession,user_id: int):
    # 生成Token + 设置过期时间 -> 查询数据库当前用户是否有Token -> 有：更新 ； 没有：添加
    token = str(uuid.uuid4())
    # timedelta(days=7,hours=2,minutes=30,seconds=10)
    expires_at = datetime.now() + timedelta(days=7)
    stmt = select(UserToken).where(UserToken.user_id == user_id)
    result = await db.execute(stmt)
    user_token = result.scalar_one_or_none()
    if user_token:
        user_token.token = token
        user_token.expires_at = expires_at
    else:
        user_token = UserToken(user_id=user_id,token=token,expires_at=expires_at)
        db.add(user_token)

    await db.commit()
    return token
```

## 补充5：token的使用
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260308110530591.png)
```python
# crud/users.py
# 根据 Token 查询用户： 验证 Token -> 查询用户
async def get_user_by_token(db: AsyncSession,token: str):
    stmt = select(UserToken).where(UserToken.token == token)
    result = await db.execute(stmt)
    user_token = result.scalar_one_or_none()
    # token不存在 或 过期
    if not user_token or user_token.expires_at < datetime.now():
        return None

    stmt = select(User).where(User.id == user_token.user_id)
    result = await db.execute(stmt)
    return result.scalar_one_or_none()
```
```python
# utils/auth.py
# 整合 根据token查询用户，返回用户
async def get_current_user(
        authorization: str = Header(...,alias="Authorization"),
        db: AsyncSession = Depends(get_db)
):
    # Authorization: Bearer <token> ——> Authorization: Bearer eyJhbGciOi
    # 从Authorization中获取token
    token =authorization.replace("Bearer ","")
    user = await users.get_user_by_token(db,token)
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED,detail="无效的令牌或已经过期的令牌")

    return user
```
```python
# routers/users.py
# 查Token、查用户 -> 封装crud -> 功能整合成一个工具函数 -> 路由导入使用: 依赖注入
@router.get("/info")
async def get_user_info(user: User=Depends(get_current_user)):
    return Result.ok(message="获取用户信息成功",data=UserInfoResponse.model_validate(user))
```

## 补充6：redis缓存功能
### （1）在Windo上安装Redis
- 在 https://github.com/tporadowski/redis/releases 网址，下载最小的 ".msi"文件进行安装
- 在cm的中验证安装是否成功：
```bash
redis-cli ping
```
- 管理 Redis 服务
```bash
# 启动服务
net start Redis

# 停止服务
net stop Redis
```

### （2）在PyCharm中 安装并配置 Redis客户端
- 安装Redis客户端
```bash
pip install redis
```
- 配置Redis客户端
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260308154733659.png)
```python
# config/cache_config.py
import redis.asyncio as redis

REDIS_HOST = "localhost"
REDIS_PORT = 6379
REDIS_DB = 0

# 创建 Redis 的连接对象
redis_client = redis.Redis(
    host=REDIS_HOST,        # Redis 服务器的主机名
    port=REDIS_PORT,        # Redis 端口号
    db=REDIS_DB,            # Redis 数据库编号,0~15
    decode_responses=True   # 是否将字节数据解码为字符串
)
```

### （3）封装缓存操作
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260308155201306.png)
```python
# config/cache_config.py
from typing import Any

import redis.asyncio as redis
from pydantic import json

REDIS_HOST = "localhost"
REDIS_PORT = 6379
REDIS_DB = 0

# 创建 Redis 的连接对象
redis_client = redis.Redis(
    host=REDIS_HOST,        # Redis 服务器的主机名
    port=REDIS_PORT,        # Redis 端口号
    db=REDIS_DB,            # Redis 数据库编号,0~15
    decode_responses=True   # 是否将字节数据解码为字符串
)

# 设置 和 读取（字符串 和 列表或字典）
# 1，读取字符串
async def get_cache(key:str):
    try:
        return await redis_client.get(key)
    except Exception as e:
        print(f"获取缓存失败：{e}")
        return None

# 2.读取：列表或字典
async def get_json_cache(key:str):
    try:
        data = await redis_client.get(key)
        if data:
            return json.loads(data) # 反序列化
        return None
    except Exception as e:
        print(f"获取 JSON 缓存失败：{e}")
        return None

# 3.设置缓存
async def set_cache(key:str,value:Any,expire:int=3600):
    try:
        if isinstance(value,(dict,list)):   # 判断类型
            # 序列化
            value = json.dumps(value,ensure_ascii=False)    # 中文正常保存
        await redis_client.setex(key,value,expire)
        return True
    except Exception as e:
        print(f"设置缓存失败：{e}")
        return False
```

### （4）设计缓存策略
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260308160211914.png)
```python
# cache/news_cache.py
# 新闻相关的缓存方法：新闻分类的读取和写入
# key - value
from typing import List, Dict, Any

from config.cache_config import get_json_cache, set_cache

CATEGORIES_KEY = "news:categories"

# 获取新闻分类缓存
async def get_cached_categories():
    return await get_json_cache(CATEGORIES_KEY)

# 写入新闻分类缓存: 缓存的数据，过期时间
# 不同类型的数据：缓存时间不一样 -> 避免所有key同时过期，引起缓存雪崩
# 分类、配置：7200   列表：600  详情：1800  验证码：120 -- 数据越稳定，缓存越持久
async def set_cached_categories(data: List[Dict[str, Any]],expire: int = 7200):
    return await set_cache(CATEGORIES_KEY,data,expire)
```
```python
# crud/get_news_cache.py
async def get_news_categories_list(db: AsyncSession,skip: int = 0, limit: int = 100):
    # 先尝试从缓存中获取数据
    cached_categories = await get_cached_categories()
    if cached_categories:
        return cached_categories

    # 未命中，则从数据库中获取数据
    stmt = select(NewsCategory).offset(skip).limit(limit)
    result = await db.execute(stmt)
    categories = result.scalars().all() # ORM对象

    # 写入缓存
    if categories:
        categories = jsonable_encoder(categories)    # 将ORM对象转换成JSON，才能缓存
        success = await set_cached_categories(categories)

    # 返回数据
    return  categories
```
```python
# router/news.py
@router.get("/categories")
async def get_news_categories(db :AsyncSession = Depends(get_db),skip: int = 0, limit: int = 100):
    # 获取数据库里面新闻分类数据：先定义模型类，在封装查询数据的方法
    # categories = await news.get_news_categories_list(db,skip, limit)
    categories = await news_cache.get_news_categories_list(db, skip, limit)
    return {
        "code": 200,
        "message": "获取新闻分类成功",
        "data": categories
    }
```

## 补充7：AI问答功能
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/e395ac1c3e0bf97c98aaa4e88d8d4463.jpg)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260308152029909.png)