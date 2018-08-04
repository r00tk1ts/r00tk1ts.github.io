---
title: quicksort
date: 2017-08-27 20:50:11
categories: algorithm
tags:
	- algorithm
---
早年写过的快排的一点总结。主要参考《算法导论》，融入自己的一些理解。

<!--more-->

# 快速排序 #
快排的最坏时间复杂度是Θ(n^2),尽管如此，快排依然是比较排序中最好的算法。它的平均复杂度为Θ(nlgn)，且常数因子尤其小。

快排和归并排序相似，核心思想都是分治。对于一个数组A[p..r]：
Divide：将A[p..r]分成两部分，A[p..q-1]和A[q+1..r]，保证A[p..q-1]中的元素都比A[q]小，A[q+1..r]中的元素都比A[q]大。
Conquer：对A[p..q-1]和A[q+1..r]递归。
Combine：null

伪算法
```
QUICKSORT(A,p,r)
if p < r
	q = PARTITION(A,p,r)
	QUICKSORT(A,p,q-1)
	QUICKSORT(A,q+1,r)

PARTITION(A,p,r)
x = A[r]
i = p - 1
for j = p to r-1
	if A[j] <= x
		i = i + 1
		exchange A[i] with A[j]
exchange A[i+1] with A[r]
return i+1
```

PARTITION可能不是那么一目了然，一图解惑：

![](/images/algorithm/partition.jpg)

显然，PARTITION对A[p..r]的时间复杂度为Θ(n)，这里n=r-p+1。

## 最坏情况的分析 ##
如果A一开始就是严格有序的，而PARTITION中我们看到每次都是选最后一个元素为主元。这样一来，每次QUICKSORT(A,p,r)时，都是将A[r]换到A[p]或者什么都不干，每次PARTITION复杂度为Θ(n)，递归下去时间复杂度也就变成了：
T(n) = T(n-1) + T(0) + Θ(n)
而T(0) = Θ(1)（显然的，array size为0直接返回了），T(n)=T(n-1)+Θ(n)。

这样一个数列，累加求和：
T(n) = T(n-1) + Θ(n)
T(n-1) = T(n-2) + Θ(n-1)
T(n-2) = T(n-3) + Θ(n-2)
...
T(1) = T(0) + Θ(0)

等差数列求和，时间复杂度也就是Θ(n^2)。

## 最优情况分析 ##
最优情况下，显然就是每次的divide都是五五开。
时间复杂度就是：T(n) = 2T(n/2) + Θ(n)

根据主方法，T(n)=aT(n/b)+f(n)，a为2，b为2，f(n)为Θ(n)。
查case：

![](/images/algorithm/mainfunc.jpg)

符合case2，所以T(n)=Θ(nlgn)

>这里的最优和最劣的情况怎么看都是依靠直觉，虽然最后的结果是正确的，从数学的角度出发，所谓的最优和最劣其实可以这样表示：
最劣：T(n) = max(T(q)+T(n-q-1))+Θ(n) (0<=q<=n-1)
最优：T(n) = min(T(q)+T(n-q-1))+Θ(n) (0<=q<=n-1)
对于某个常数c，我们假设T(n)<=cn^2，那么带入得到：
T(n) = c·(q^2+(n-q-1)^2)+Θ(n) （0<=q<=n-1）
由均值不等式知，最大值在边缘取到，最小值在n-q-1=q取到。
最劣：T(n) <= cn^2 - c(2n-1) + Θ(n) <= cn^2(我们可以取到一个足够大的c让c(2n-1)制约掉Θ(n))，所以满足猜想，T(n)=O(n^2)。
最优：2c((n-1)/2)^2+Θ(n) = c/2 * (n-1)^2，显然也是满足猜想，但这并不严格。且我们已经得到了条件，只需要求T(n) = 2T(n/2) + Θ(n)即可，主方法已经搞得定了，没必要画蛇添足。

## 平均情况分析 ##
假设分组情况如下，PARTITION总是产生9:1的划分：


![](/images/algorithm/partition-average.jpg)

可以看出分组多的那一侧决定了树的高度。而对于每一层而言，PARTITION的处理不会超过cn，而树不会超过lgn层（常数小的那个），那么显然时间复杂度为O(nlgn)。

>关于平均情况的复杂度，可参考算法导论的严格证明，但是其表述较为麻烦，这里给出知乎“芾棠”的简易方法：

![](/images/algorithm/quicksort-average.jpg)

## 随机化主元选取 ##
显然，为了保证我们的主元选取尽可能的满足平均情况（即避开恶意构造有序数组的情况），我们来个类似加密手法中加salt的手段：每次主元选取时都随机化。

伪算法
```
RANDOMIZED-PARTITION(A,p,r)
i = RANDOM(p,r)
exchange A[r] with A[i]
return PARTITION(A,p,r)

RANDOMIZED-QUICKSORT(A,p,r)
if p < r
	q = RANDOMIZED-PARTITION(A,p,r)
	RANDOMIZED-QUICKSORT(A,p,q-1)
	RANDOMIZED-QUICKSORT(A,q+1,r)
```

## [CODE]quicksort by C++ ##
```cpp
/* quicksort.h */
#include <iostream>
#include <vector>
#include <iterator>
#include <ctime>
#include <algorithm>

using namespace std;

namespace algo
{
	void QuickSort(vector<int> &vec, int begin, int end);
}

/* quicksort.cpp */
#include "quicksort.h"

namespace algo
{
	void QuickSort(vector<int> &vec, int begin, int end)
	{
		if(begin >= end)
			return ;
		int random = (rand()%(end - begin + 1)) + begin;
		swap(vec[random],vec[end]);

		int i = begin;
		for(int j=begin;j!=end;j++)
		{
			if(vec[j] < vec[end])
				swap(vec[i++],vec[j]);
		}

		swap(vec[i],vec[end]);

		QuickSort(vec,begin,i-1);
		QuickSort(vec,i+1,end);
	}
}

/* main.cpp */
#include "quicksort.h"

void testQuickSort()
{
	vector<int> vec;
	for(int i=0;i<100;i++)
	{
		vec.push_back(rand());
	}
	cout << "随机填充100个数" << endl;
	copy(vec.begin(),vec.end(),ostream_iterator<int>(cout,"\t"));
	algo::QuickSort(vec,0,vec.size()-1);
	cout << "快排" << endl;
	copy(vec.begin(),vec.end(),ostream_iterator<int>(cout, "\t"));
}

int main()
{
	testQuickSort();
	return 0;
}
```

测试结果：

![](/images/algorithm/quicksort-result.jpg)