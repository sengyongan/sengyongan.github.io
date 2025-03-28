---
title: 1`树状数组(Binary Indexed Tree -- BIT)
date: 2024-11-07 12:00:00 +0800
categories: [算法与数据结构]
tags: []     # TAG names should always be lowercase
math: true
---
# 树状数组(Binary Indexed Tree -- BIT)

BIT：给定1维数组映射（lowbit）为树形结构
**（注：此图为转载）**
![1730981732408](/assets/img/blog/other/树状数组.png)

## lowbit

lowbit（x）函数：非负整数n 的二进制形式 1的最低位以及后面0 组成的数
例：lowbit（44）= lowbit(10110) = 100(2进制) = 4（10进制）
lowbit(x) = x & (-x)

## 树形结构

根据lowbit建立树形结构父子关系 -> x的父节点 = x + lowbit(x) (注：x为1---n，即数组序号)
其中每个树形结构节点t[x]的值为以t[x]为根树的叶节点和

## BIT作用--前缀查询

为什么要将1维数组映射到这样的树形结构呢？
可以在logN的时间，计算前缀和 / 区间和
比如：如果要计算sum(7)->a[1]--a[7]的和

```c++
-> = t[7]+t[6]+t[4], 
而 6 = 7 - lowbit(7), 4 = 6 - lowbit(6) 
= t[7] + t[（7-lowbit(7)）] + t[（（7-lowbit(7)）-lowbit(（7-lowbit(7)）) ]
```

如果没有树形结构，那么我们要遍历7个数组元素求和，而现在只需要计算3次，即可获得前7个元素的和

```c++
int ask(x){
	int sum = 0;
	for(int i=x;i;i-=lowbit(i)){
		sum+=t[i];
	}
	return sum;
}
```

## BIT--单点修改

为了维护树形数组，当我们修改任意一个a[x]的值，即a[x] + k，都要修改所有t[x] + k以及所有父节点的值 + k

```c++
int add_dandian(int x,int k)
{
	for(int i=x;i<=n;i+=lowbit(i))
	t[i]+=k;
}
```

## BIT作用--区间查询

如何求[L,R]的区间和呢？
利用前缀和相减的性质：[ L , R ] = [ 1 , R ] − [ 1 , L − 1 ]

```c++
int search(int L,int R)
{
	int ans = 0;
	for(int i=L-1;i;i-=lowbit(i))
	ans-=c[i];
	for(int i=R;i;i-=lowbit(i))
	ans+=c[i];
	return 0;
}
```

## BIT--区间修改

对区间[L,R]+k

```c++
int update(int pos,int k)
{
	for(int i=pos;i<=n;i+=lowbit(i))
	c[i]+=k;
	return 0;
}
update(L,k);
update(R+1,-k);
```

## 实例

> ***面试题 10.10. 数字流的秩***
> 假设你正在读取一串整数。每隔一段时间，你希望能找出数字 x 的秩(小于或等于 x 的值的个数)。请实现数据结构和算法来支持这些操作，也就是说：
> 实现 track(int x) 方法，每读入一个数字都会调用该方法；
> 实现 getRankOfNumber(int x) 方法，返回小于或等于 x 的值的个数
> x <= 50000

也就是如何快速找到前缀和，track就是单点修改，getRankOfNumber就是前缀查询

```c++
class StreamRank {
    vector<int> vec;
    int lowbit(int x) {
        return (x & (-x));
    }
    /* 
        原数据是0--50000，BIT是1--50001，因此vec应有50002个，即从0--50001
    */
public:
    StreamRank() : vec(50002, 0){}
    void track(int x) {
        ++x;//映射到从1开始的BIT中
        while(x <= 50001){
            vec[x]++;
            x += lowbit(x);
        }
    }
    int getRankOfNumber(int x) {
        ++x;//映射到从1开始的BIT中
        int res;
        while(x){
            res += vec[x];
            x -= lowbit(x);
        }
        return res;
    }
};
```
