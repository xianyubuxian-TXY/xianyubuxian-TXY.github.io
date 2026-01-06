---
title: N皇后问题
date: 2026-01-05 22:09:01
tags:
	- oj
	- 笔试
categories: 
	- 编程题
	- 递归/回溯
---

# 1.问题描述
![题目描述](/images/N皇后问题.png)

# 2.问题分析：
	思路很直接：按行放皇后，每放一个就标记该列和两条对角线被占用，放完最后一行时计数＋＋，然后回溯撤销。

# 3.解决方法：递归+回溯
## (1)利用矩阵的性质
	1./ 方向的所有格子满足 i+j == 常量
	2.\ 方向的所有格子满足 i−j == 常量，为了下标非负，偏移了 +(n−1) ——>注意：不可以简单的用绝对值，如（1,2），(2,1)
```cpp
    void backtrace(int& res,int row,vector<bool>& col,vector<bool>&diag1,vector<bool>&diag2,int n){
        if(row==n){
            res++;
            return;
        }
        //遍历每一列
        for(int j=0;j<n;++j){
            //检查所在列、对角线是否还可用
            //关键点1：“/斜线上：row+col=定值”，“\斜线上：row-col=定值——>为保持非负，向右偏移n-1（注意不可以简单的用“绝对值”）”
            if(col[j] || diag1[row+j] || diag2[row-j+n-1]) continue;
            //可用：占用并更新col、diag1、diag2
            col[j]=diag1[row+j]=diag2[row-j+n-1]=true;
            //递归遍历下一行
            backtrace(res,row+1,col,diag1,diag2,n);
            //关键点2：回溯
            col[j]=diag1[row+j]=diag2[row-j+n-1]=false;
        }
    }
    
    int Nqueen(int n) {
        // write code here
        //col：同列是否可用   diag1：/对角线是否可用  diag2：\对角线是否可用  ->false:未占用  true:占用
        vector<bool> col(n,false),diag1(n*2,false),diag2(n*2,false);
        int res=0;
        backtrace(res,0,col,diag1,diag2,n);
        return res;
    }
```

## (2)逐行记录已放皇后的列，并每次遍历已放皇后检查冲突（如果不知道上述矩阵的性质）
```cpp
    void backtrace(int& res,int row,vector<int>& pos,int n) {
        if (row == n) {
            res++;
            return;
        }
        // 尝试把第 row 行的皇后放到 col 列
        for (int col = 0; col < n; ++col) {
            bool ok = true;
            // 检查和前面所有行的冲突
            for (int r = 0; r < row; ++r) {
                // 同列冲突 或 同对角线冲突（|行差|==|列差|）
                if (pos[r] == col || abs(row - r) == abs(col - pos[r])) {
                    ok = false;
                    break;
                }
            }
            if (!ok) continue;
            // 放置皇后
            pos[row] = col;
            backtrace(res, row + 1, pos, n);
            // 回溯时不需要手动清理 pos[row]，下次覆盖即可
        }
    }

    int Nqueen(int n) {
        int res = 0;
        vector<int> pos(n, -1); //记录每行放置在哪一列
        backtrace(res, 0, pos, n);
        return res;
    }
```