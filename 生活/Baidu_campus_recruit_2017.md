---
date: 2016-08-26 23:28
status: public
tags: Algorithms
title: 百度实习生招聘笔试题
---

# 1 描述
给两个二叉树，判断第二个二叉树是否为第一棵二叉树的字数，并且二叉树节点的值没有重复的。

二叉树节点定义：
```C++
typedef struct tnode{
    int val;
    struct tnode* left;
    struct tnode* right;
};

int isSubtree(tnode* root1, tnode* root2)
{
//todo
}
```
# 2 解题思路
在题意中有一个二叉树节点没有重复，则可以考虑到二叉树重建的实现：任意一个二叉树，给定二叉树[先序|后序] && 中序即可重建这棵二叉树，LeetCode上有相关的题目。
+ 构建两棵树的先序遍历和中序遍历数据 
+ 判断第二个二叉树的先序遍历是否为第一个二叉树先序遍历的子序列
+ 再判断第二个二叉树的中序遍历是否为第一个二叉树中序遍历的子序列 
+ 由于二叉树的节点的值唯一，所以一旦匹配则表明满足题意。匹配的过程可采用KMP算法，提高算法效率 
+ 遍历算法可以采用递归也可以采用迭代方式 


# 3 代码 
```C++
#include <vector>
using namespace std;
typedef struct tnode{
    int val;
    struct tnode* left;
    struct tnode* right;
};
void preOrder(tnode* node, vector<int>& vec)
{
    if (node != nullptr)
    {
        vec.push_back(node->val);
        preOrder(node->left, vec);
        preOrder(node->right, vec);
    }
}
void inOrder(tnode* node, vector<int>& vec)
{
    if (node != NULL)
    {
        inOrder(node->left, vec);
        vec.push_back(node->val);
        inOrder(node->right, vec);
    }
}
int* buildNext(vector<int>& pattern)
{
    size_t m = pattern.size(), j = 0;
    int* N = new int[m];
    int t = N[0] = -1;
    while (j<m - 1)
    {
        if (0>t || pattern[j] == pattern[t])
        {
            j++;
            t++;
            N[j] = t;
        }
        else
            t = N[t];
    }
    return N;
}
bool isMath(vector<int>& vec1, vector<int>& vec2)
{
    int* next = buildNext(vec2);
    int n = vec1.size(), i = 0;
    int m = vec2.size(), j = 0;
    while (j < m&&i < n)
    {
        if (0>j || vec1[i] == vec2[j])
        {
            i++;
            j++;
        }
        else
            j = next[j];
    }
    delete[] next;
    return (i - j) != n;
}

int isSubtree(tnode* root1, tnode* root2)
{
    vector<int> pre1, pre2;
    vector<int> in1, in2;
    tnode* node1 = root1;
    tnode* node2 = root2;
    preOrder(node1, pre1);
    preOrder(node2, pre2);
    node1 = root1;
    node2 = root2;
    inOrder(node1, in1);
    inOrder(node2, in2);

    bool res = true;
    res &= isMath(pre1, pre2);
    if (res == false)
        return 0;
    res&= isMath(in1, in2);
    if (res)
        return 1;
    else
        return 0;
}
```