---
title: 2.设计LFU缓存结构
date: 2026-03-24 11:47:46
tags:
	- oj
	- 笔试
categories:
	- 编程题
	- 模拟
---

# 1.题目描述
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260324114842900.png)

# 2.题目解析
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260324114935783.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260324115124334.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260324115157902.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260324115212566.png)
![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260324115256662.png)![](https://cdn.jsdelivr.net/gh/xianyubuxian-TXY/hexo-images/images/20260324115350599.png)


# 3.代码实现
```cpp
#include <queue>
#include <unordered_map>
class Solution {
public:
    /**
     * 代码中的类名、方法名、参数名已经指定，请勿修改，直接返回方法规定的值即可
     *
     * lfu design
     * @param operators int整型vector<vector<>> ops
     * @param k int整型 the k
     * @return int整型vector
     */
    vector<int> LFU(vector<vector<int> >& operators, int k) {
        // write code here
        vector<int> res;
        int len=operators.size();
        for(int i=0;i<len;i++){
            int op=operators[i][0];
            if(op==1){  // set操作
                int opKey=operators[i][1];
                int opVal=operators[i][2];
                auto it=umap_.find(opKey);
                if(it!=umap_.end()){    // 记录存在，更新值
                    auto [val,freq,time]=umap_[opKey];   // 解构tuple
                    
                    // 从set_中删除该记录
                    set_.erase({freq,time,opKey});
                    
                    // 更新该该记录，并修改umap_,插入set_
                    freq++;
                    time++;
                    umap_[opKey]={opVal,freq,time};   // 更新umap_中对应的值

                    set_.insert({freq,timer,opKey});
                }else{  // 记录不存在
                    int freq=1;
                    timer++;
                    if(umap_.size()<k){ // 容量未满，直接插入
                        umap_.insert({opKey,{opVal,freq,timer}});
                        set_.insert({freq,timer,opKey});
                    }else{  // 容量以满，删除set_最前面的记录，再插入
                        auto [freq,time,key]=*set_.begin();
                        set_.erase({freq,time,key});    // 从set_中删除
                        umap_.erase(key);   // 从umap_中删除

                        set_.insert({freq,timer,opKey});   // 将新纪录插入set_
                        umap_.insert({opKey,{opVal,freq,timer}});
                    }
                }
            }else{  // get操作
                int key=operators[i][1];
                auto it=umap_.find(key);
                if(it==umap_.end()) res.push_back(-1);  // 不存在，返回值为-1
                else{   // 存在，获取值后需要更新set_
                    auto [val,freq,time]=umap_[key];
                    res.push_back(val);

                    // 从set_中删除旧值
                    set_.erase({freq,time,key});    
                    // 更新记录状态
                    freq++;
                    timer++;

                    umap_[key]={val,freq,timer}; // 更新umap_
                    set_.insert({freq,timer,key});   // 插入set
                }
            }
        }

        return res;
    }

    int timer=0;    // 计时器

    // key ——> <val,freq,time>
    unordered_map<int,tuple<int,int,int>> umap_;

    // <fre,time,key> 自动按字典序升序排列
    set<tuple<int,int,int>> set_;
};
```
