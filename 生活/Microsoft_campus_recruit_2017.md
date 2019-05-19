---
date: 2016-10-11 19:18
status: public
tags: Algorithms
title: Microsoft笔试
---

#  1  Shortening Sequence
> 描述
    There is an integer array A1, A2 ...AN. Each round you may choose two adjacent  integers. If their sum is an odd number, the two adjacent integers can be deleted.
Can you work out the minimum length of the final array after elaborate deletions?
输入
The first line contains one integer N, indicating the length of the initial array.
The second line contains N integers, indicating A1, A2 ...AN.
For 30% of the data：1 ≤ N ≤ 10
For 60% of the data：1 ≤ N ≤ 1000
For 100% of the data：1 ≤ N ≤ 1000000, 0 ≤ Ai ≤ 1000000000
输出
One line with an integer indicating the minimum length of the final array.
样例提示
(1,2) (3,4) (4,5) are deleted.  

## 解题思路  
题目的大意就是给定一个数组，以此删除相邻的数字，当数字之和等于奇数时候，这两个数据同时删除。这个过程可以重复进行，直至无法删除任何数据，求解出剩余的数字的个数。 
## 算法
1. 根据栈的工作性质可以缓冲操作，当读入一个数据的时候，跟栈顶数据相加，如果为奇数，POP操作；如果为偶数，入栈。
```C#
static void Process(string line)
{
    Stack<int> stack=new Stack<int>(line.Length);
    string[] tokens=line.Split(' ');
    for(int i=0;i<tokens.Length;i++)
    {
        if(stack.Length==0)
            stack.Push(int.Parse(tokens[i]));
        else
        {
            int top=stack.Top();
            int current=int.Parse(tokens[i]);
            if((top+current)%2==0){
                stack.Push(top);
            }else{
                stack.Pop();
            }
        }
    }
    Console.WriteLine(tokens.Length-stack.Count);
}
```   
 
2. 通过简单的数论只是可知：偶数+奇数=奇数。因此最多匹配对最大值所有奇数和偶数最小值
```C#
static void Process(string line)
{
    string[] tokens=line.Spilt(' ');
    int odd=0;
    int even=0;
    for(int i=0;i<token.Length;i++)
    {
        int value=int.Parse(token[i]);
        if(value%2==0)
            even++;
        else
            odd++;
    }
    Console.WriteLine(token.Length-Math.Min(odd,even)*2);
}
```

# Composition 
> 描述
Alice writes an English composition with a length of N characters. However, her teacher requires that M illegal pairs of characters cannot be adjacent, and if 'ab' cannot be adjacent, 'ba' cannot be adjacent either.
In order to meet the requirements, Alice needs to delete some characters.
Please work out the minimum number of characters that need to be deleted.
输入
The first line contains the length of the composition N.
The second line contains N characters, which make up the composition. Each character belongs to 'a'..'z'.
The third line contains the number of illegal pairs M.
Each of the next M lines contains two characters ch1 and ch2,which cannot be adjacent.
For 20% of the data: 1 ≤ N ≤ 10
For 50% of the data: 1 ≤ N ≤ 1000
For 100% of the data: 1 ≤ N ≤ 100000, M ≤ 200.
输出
One line with an integer indicating the minimum number of characters that need to be deleted.  
## 算法 
### 回溯法
从头开始扫描整个字符串，判断其相邻的字符串是否为Illegal字符串，如果是，删除其中一个字符串，生成两个字符串，分别回溯求解，求解最小值。
```C#
static int DeleteCharacters(string str,int start,ISet<String> illeage)
{
    if(string.IsNullOrEmpty(str)|| start>=str.Length-1) return 0;
    int j=start;
    for(;j<str.Length-1;j++)
    {
        string pair=str[j]+str[j-1].ToString();
        if(illeage.Contains(pair)) break;
    }
    if(j==str.Lenght-1) return 0;
    string first=DeleteOneCharacter(str,j);
    string second=DeleteOneCharacter(str,j+1);
    return 1+Math.Min(DelteCharacter(first,j==0?0:j-1,illeage),
        DelteCharacter(second,j,illeage);
}
static string DeleteOneCharacter(string str,int index)
{
    StringBuilder sb=new StringBuilder(str.Length-1);
    for(int i=0;i<str.Length;i++){
        if(i!=index)sb.Append(str[i]);
    }
    return sb.ToString();
}
```   

然而该算法LTE。  
### Dynamic programming 
由于已知所有的字符串均为a-z, 所以采用dp[i]表示第i+1个字符结尾，最长的字符串的长度。
```C#
static void Process(string line,ISet<string> set)
{
    int[] dp=new int[line.Length];
    for(int i=0;i<line.Length;dp[i++]=0);
    for(int i=1;i<line.Length.Length;i++)
    {
        int temp=dp[i];
        for(int j=0;j<i;j++){
            int index=j;
            while(index>=0&&set.Contains(line[index]+line[i].ToString()){
                index--;
            }
            temp=index==-1?Math.Max(temp,1):Math.Max(temp,dp[index]+1);
        }
        dp[i]=temp;
    }
    Console.WriteLine(line.Length-dp[line.Length-1]);
}
```