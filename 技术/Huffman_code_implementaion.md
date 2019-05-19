---
date: 2016-08-20 18:08
status: public
tags: Algorithms
title: Huffman编码树实现
---

#定义
在Huffman编码树是基于加权最短路径树，具体定义见：[Huffman Tree](https://en.wikipedia.org/wiki/Huffman_coding) 
#实现过程
1. 输入各个字符已经相应的字符权重
2. 构建Huffman编码树节点向量
3. 在向量中查找权重最小的两个节点，新建一个新的节点，其左右子树是两节点，权重为两个子树权重之和
4. 添加到向量中，并删除两个字数
5. 重复3-4，直到只有一个节点  

#注意点
1. 查找两个最小节点的采用的是 Divide and Conquer ，算法复杂度为（logN)， 如果采用遍历的方法算法复杂度达（2n).  将向量均分成两部分，分别求得每个部分的最小的两个Tuple1（Min1, Min2),和Tuple2(Min1,Min2), 将比较这两个部分的最小的两个，递归基为向量的个数小于4个
```C#
/// <summary>
       /// 递归基，平凡情况，如果只有两个或者三个要素时
       /// </summary>
       /// <param name="huffchars"></param>
       /// <param name="lo"></param>
       /// <param name="hi"></param>
       /// <returns></returns>
       private  Tuple<Int32, Int32> TrivialTwoMin(IVector<BinNode<HuffChar>> huffchars, Int32 lo, int hi)
       {
           int first = huffchars[lo].Data.Weight < huffchars[lo + 1].Data.Weight ? lo : lo + 1;
           int second = first == lo ? lo + 1 : lo;
           for (int i = lo + 2; i < hi; i++)
           {
               if (huffchars[i].Data.Weight < huffchars[second].Data.Weight)
               {
                   if (huffchars[i].Data.Weight < huffchars[first].Data.Weight)
                   {
                       second = first;
                       first = i;
                   }
                   else
                   {
                       second = i;
                   }
               }
           }
           return new Tuple<int, int>(first, second);
       }

       /// <summary>
       /// 递归迭代版查找最小的两个要素
       /// </summary>
       /// <param name="huffchars"></param>
       /// <param name="lo"></param>
       /// <param name="hi"></param>
       /// <returns></returns>
       private  Tuple<Int32, Int32> FindTwoMin(IVector<BinNode<HuffChar>> huffchars, Int32 lo, Int32 hi)
       {
           if (hi - lo <= 3)
           {
               return TrivialTwoMin(huffchars, lo, hi);
           }
           int mi = (hi + lo) >> 1;
           Tuple<Int32, Int32> firstPart = FindTwoMin(huffchars, lo, mi);
           Tuple<Int32, Int32> secondPart = FindTwoMin(huffchars, mi, hi);
           if (huffchars[firstPart.Item1].Data.Weight < huffchars[secondPart.Item1].Data.Weight)
           {
               return huffchars[firstPart.Item2].Data.Weight < huffchars[secondPart.Item1].Data.Weight ?
                   firstPart :
                   new Tuple<int, int>(firstPart.Item1, secondPart.Item1);
           }
           return
               huffchars[secondPart.Item2].Data.Weight < huffchars[firstPart.Item1].Data.Weight ?
                   secondPart :
                   new Tuple<int, int>(secondPart.Item1, firstPart.Item1);
       }
``` 
2. 获取各个叶节点的编码。遍历整个二叉树，判断如果当前节点为叶节点，通过Parent指针一直到根节点，左孩子为0，右孩子为1，获取每个字符串的编码 
    
```C#
/// <summary>
       /// 遍历整个二叉树
       /// </summary>
       private void BuildCodeMap()
       {
           _huffmanRoot.TravIn(BuildCodeMap);
       }

       /// <summary>
       /// 如果是叶节点
       /// </summary>
       /// <param name="node"></param>
       private void BuildCodeMap(BinNode<HuffChar> node)
       {
           if (!node.HasBothChild)
           {
               char code = node.Data.Ch;
               string s = string.Empty;
               while (node!=_huffmanRoot)
               {
                   if (node.IsLChild)
                   {
                       s += "0";
                   }
                   else
                   {
                       s += "1";
                   }
                   node = node.Parent;
               }
               _charCodeMap.Add(code,new string(s.Reverse().ToArray()));
           }
       }
``` 
#代码实现
[github](https://github.com/gaufung/DS)