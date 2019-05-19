---
date: 2016-10-22 16:04
status: public
title: MIT算法导论课程学习5
---

# Chapter13
**Mining Spanning Tree**
对于无向带权图（undirected weighted graph），n为顶点数，选择n-1条边，包含这n个顶点。使之成为一个支撑树，而且树边权值和最小。
## Prim Algorithm
Prim 算法将图中的所有节点划分为支撑树节点集合A和其余节点集合V-A，每次从两个集合中割中最小堆P中选择代价最小边E[i,j],将集合V-A中的顶点加入到集合A中。该算法每次添加到的集合中选择都是最小边，因此为**贪心**算法。
算法输入时连通图G和最小生成树的根r。不在树中的顶点放在一个基于key域的最小优先级队列Q中，key[v]是所有将v与树中某一顶点相连的边中的最小权值，如果不存在key[v]=∞，p[v]是指树中v的父母、
```
MST-PRIM(G,w,r)
for each u ∈ V[G]
    do key[u] <- ∞
        p[u] <- nil
key[r] <- 0
Q <- V[G]
while Q≠∅
    do u <- extract-min(Q)
        for each v ∈ Adj[u]
            do if v ∈ Q and w(u,v) < key[v]
                then p[v] <- u
                    key[v] <- w(u,v)
```
时间复杂度: $\Theta(VlgV+ElgV)=\Theta(ElgV)$ 

## Kruskal算法
Krushkal算法与Prim算法相反，关注的是边，先对所有边进行非递减的排序，并建立V个子树，依次选择最小权重的边(u,v)，如果边端点u和v不属于同一棵子树，将两棵子树合并成一个新的子树，反之舍弃之。
```
MST-KRUSKAL(G,w)
A <- ∅
for each vertex v ∈ V[G]
    do MAKE-SET(v)
sort the edge of E into nondecreasing order by weight w
for each edge(u,v) ∈ E， token in nondecreasing order by weight
    do if FIND-SET(u)≠Find-SET(v)
        then A <- A ∪ {(u,v)}
            UNION(u,v)
return A
```
时间复杂度： $\Theta(ElgV)$