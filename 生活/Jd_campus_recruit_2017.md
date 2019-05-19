---
date: 2016-09-05 21:30
status: public
tags: 生活
title: 京东2017年校招(1)
---

投递的是京东数据挖掘岗，其中编程大题大体是这样的：
给定三个圆(x,y,r),在平面中求解一个点，使得该点与每一个圆的两个外切线组成的角度相等。
#  0 分析
这是一道平面几何题目，转换成数学语言就是$sin(\frac{\theta}{2})=\frac{r}{distance(O,P)}$,对于三个圆而言$sin(\frac{\theta_{1}}{2})=sin(\frac{\theta_{2}}{2})=sin(\frac{\theta_{1}}{2})$。
$$\begin{equation} \begin{cases} (x-x_{1})^{2}+(y-y_{1})^{2}=r_{1}^{2}\alpha \\\ (x-x_{2})^{2}+(y-y_{2})^{2}=r_{2}^{2}\alpha \\\ (x-x_{3})^{2}+(y-y_{3})^{2}=r_{3}^{2}\alpha \end{cases} \end{equation}$$
共有三个参数并且有三个方程，(1)-(2)和(2)-(3)可得：
$$\begin{equation} \begin{cases} 2a_{1}x+2b_{1}y+c_{1} = \alpha d_{1} \\\ 2a_{2}x+2b_{2}y+c_{2}=\alpha d_{2}\end{cases} \end{equation} $$
其中
$$ \begin{equation} \begin{cases} a_{1}=x_{2}-x_{1} \\\ a_{2}=x_{3}-x_{2} \\\ b_{1}=y_{2}-y_{1} \\\ b_{2}=y_{3}-y_{2} \\\ c_{1}=x_{1}^{2}+y_{1}^{2}-(x_{2}^{2}+y_{2}^{2}) \\\ c_{2}=x_{2}^{2}+y_{2}^{2}-(x_{3}^{2}+y_{3}^{2}) \\\ d_{1}=r_{1}^{2}-r_{2}^{2} \\\ d_{2}=r_{2}^{2}-r_{3}^{2}  \end{cases} \end{equation} $$
继续求解：
$$\begin{equation} \begin{cases} y=h\alpha +i  \\\ x=j\alpha +k \end{cases} \end{equation}$$
其中$e=2b_1a_2-2b_2a_1,f=c_1a_2-c_2a_1,g=d_1a_2-d_2a_1,h=\frac{g}{e},i=\frac{-f}{e},j=\frac{d_1-2b_1h}{2a_1},k=-\frac{2b_1i+c_1}{2a_1}$
再带入原始方程得到
$$A \alpha^{2}+B \alpha +C = 0$$
其中$A=j^2+h^2,B=2j(k-x_1)+2h(i-y_1)-r_1^2,C=(k-x_1)^2+(i-y_1)^2$
如果$A \eq 0$，则方程为一元一次方程，结果为$-\frac{C}{B}$,判断结果是否为大于1。
若$A\ne0$按照一元二次方程求根公式，将会得到三个结果：
+   无解：输出No。
+   两个相同的实数解并且$\alpha > 1$：输出位置。
+   两个不同的实数解并且$\alpha >1 $：分别计算，并且求得角度最大位置的解。  

# 1 代码实现  
```python 
import math
def parseArgument(line):
    arguments=line.split(' ')
    return float(arguments[0]),float(arguments[1]),float(arguments[2])

def calc(A,B,C):
    if(A==0.0):
        return -1*C/B
    delta=B*B-4*A*C
    if(delta<0):
        return delta
    elif delta==0.0:
        return -1*B/(2*A)
    else:
        result1=(-1*B+math.sqrt(delta))/(2*A)
        result2=(-1*B-math.sqrt(delta))/(2*A)
        if(result1<=1.0):
            return result1
        elif(result2>1.0):
            return result2
        else:
            return result1
        

def percase(line):
    x1,y1,r1=parseArgument(line)
    x2,y2,r2=parseArgument(line=input())
    x3,y3,r3=parseArgument(line=input())
    a1=x2-x1
    b1=y2-y1
    c1=(x1*x1+y1*y1)-(x2*x2+y2*y2)
    d1=r1*r1-r2*r2
    a2=x3-x2
    b2=y3-y2
    c2=(x2*x2+y2*y2)-(x3*x3+y3*y3)
    d2=r2*r2-r3*r3
    e=2*b1*a2-2*b2*a1
    f=c1*a2 - c2*a1
    g=d1*a2 - d2*a1
    h=g/e
    i=-1*f/e
    j=(d1-2*b1*h)/(2*a1)
    k=-1*(2*b1*i + c1)/(2*a1)
    A=j*j + h*h
    B=2*j*(k - x1) + 2*h*(i - y1) - r1*r1
    C=(k - x1)*(k - x1) + (i - y1)*(i - y1)
    alpha=calc(A,B,C)
    if(alpha<=1.0):
        print 'no'
    else:
        x=j*alpha+k
        y=h*alpha+i
        print str(x)+' '+str(y)

while(True):
    line=input()
    if len(line) <= 0:
        break
    percase(line)

```