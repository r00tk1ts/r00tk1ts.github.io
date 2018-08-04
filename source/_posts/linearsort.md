---
title: linear sort
date: 2017-08-27 20:51:11
categories: algorithm
tags:
	- algorithm
---
早年写过的线性时间排序的一点总结。主要参考《算法导论》，融入自己的一些理解。

<!--more-->

# 线性排序 #
归并排序，堆排序，快速排序都是基于比较来定序。对于比较排序而言，其平均时间复杂度一定是O(nlgn)。究其原因，就在于比较的次数。

一沙一世界，我们看最小的比较模型所构成的决策树：

![](/images/algorithm/decision-tree.jpg)

3个元素比较的话有3!=6种结果，也就是对应树的叶子节点。而n个元素比较有n!种结果，如果对n绘制决策树的话，其最终的叶子节点应该是n!个。显然，根据我们观察的3元素模型，其最终不会是一棵满二叉树。那么现在假设最终树的高度为h，那么就不难得出n! <= 2^h

推导一下，h >= lg(n!) = Θ(nlgn)，得出我们的结论。

> 引申一下lg(n!)的上下界为什么都是nlgn，lg(n!) <= nlgn是显然的，我们证另一边。

![](/images/algorithm/lgn!.jpg)

> 上下两行首尾相乘，每一项都大于等于n,（(a+1)(n-a)>=n {1 <= a <= (n-1)}显然）。
> 所以lg(n!) <= nlgn <= 2lg(n!)，lg(n!) = Θ(nlgn)。

**而对于线性排序来说，其核心思想并不是通过比较来定序，可谓诸家修为，各有所长。线性排序的时间复杂度为Θ(n)**

## 计数排序 ##
伪算法

```
COUNTING-SORT(A,B,k)
let C[0..k] be a new array
for i=0 to k
	C[i] = 0
for j=1 to A.length
	C[A[j]] = C[A[j]] + 1
//C[i] now contains the number of elements equal to i.
for i=1 to k
	C[i] = C[i] + C[i-1]
//C[i] now contains the number of elements less than or equal to i.
for j = A.length downto 1
	B[C[A[j]]] = A[j]
	C[A[j]] = C[A[j]] - 1
```

执行过程图例

![](/images/algorithm/counting-sort.jpg)

所谓计数，就是将A中的每种可能存在的值进行统计，个数放于C中，C接着对前元素进行累加，此时C的每个元素i就表示i这个值在B中的位置不会超过C[i]的索引，而后续每放入一个i，C[i]就递减。最终完成图f的过程。

不难看出计数排序局限性是很大的，首先C的长度是一个很大的问题，如果A数组换一下，改成2,5,3,0,2,3,0,64。那么对应C的长度就要骤增到65。

时间复杂度也是显而易见的，Θ(k+n)=Θ(n)

## 桶排序 ##
桶排序和计数排序很像，都是约束了所谓的input，来实现线性时间复杂度的一种算法。
一图胜千言：

![](/images/algorithm/bucket-sort.jpg)

伪算法

```
BUCKET-SORT(A)
let B[0..n-1] be a new array
n = A.length
for i=0 to n-1
	make B[i] an empty list
for i=1 to n
	insert A[i] into list B[[nA[i]]]
for i=0 to n-1
	sort list B[i] with insertion sort
concatenate the lists B[0],B[1],...,B[n-1] together in order
```
时间复杂度上来看，除了insertion sort这一步，其余步时间复杂度为O(n)。
深入计算：

![](/images/algorithm/bucket-sort-calc.jpg)


## 基数排序 ##
基数排序不像是一种排序，而是一种多组排序的手法。
伪算法

```
RADIX-SORT(A,d)
for i=1 to d
	use a stable sort to sort array A on digit i
```

一组数，从个位到最高位，按照每个维度进行子排序，而这个子排序必须是稳定的，也就是说，对于每一次排序，权重一致的任意两个数需要保持原本的顺序而不能互换。

直观上可能MSD更符合排序的想法，但是实际操作一下就会发现，LSD更符合基数排序的论调。
上面伪算法的描述就是LSD的应用。子排序算法可以用计数排序。

![](/images/algorithm/radix-sort.jpg)

时间复杂度：如果子排序的复杂度为Θ(n+k)，那么维度也就是最大位数为d的一组元素基数排序时间复杂度就是Θ(d(n+k))。

实际上MSD也完全可以，这就是另一种基数排序。这种排序方法类似桶排序，首先根据最高位进行分桶，分出来的每个桶再按次高位递归下去。分而治之，最终combine起来的数组就是有序的。而看到这里也就发现，如果单纯的按照最高位分桶再采取诸如insertion sort，quicksort等子排序算法，其实就是桶排序。

## [CODE]linearsort by C++ ##

```cpp
/* linearsort.h */
#include <iostream>
#include <algorithm>
#include <vector>
#include <iterator>
#include <cmath>

namespace algo
{
	void countingSort(std::vector<int> &v, int const maxNum, std::vector<int> &sorted);
	void bucketSortRec(std::vector<int> &v, int digit, std::vector<int> &sorted);
	void bucketSort(std::vector<int> &v, int digit, std::vector<int> &sorted);
	void radixSort(std::vector<int> &v, std::vector<int> &sorted, std::string &method=std::string("lsd"));
}

/* linearsort.cpp */
#include "linearsort.h"

namespace algo
{
	int getDim(int num, int digit)
	{
		int ret = num / (int)pow(10.0,digit) % 10;
		return ret;
	}

	void countingSort(std::vector<int> &v, int const maxNum, std::vector<int> &sorted)
	{
		std::vector<int> c(maxNum,0);
		std::for_each(v.begin(),v.end(),[&](int i)
		{
			++c[i];
		});

		for(int i=0;i<maxNum;i++)
		{
			for(int j=0;j<c[i];j++)
			{
				sorted.push_back(i);
			}
		}	
	}

	void bucketSortRec(std::vector<int> &v, int digit, std::vector<int> &sorted)
	{
		if(digit == -1)
		{
			for(std::vector<int>::iterator iter=v.begin();iter!=v.end();iter++)
				sorted.push_back(*iter);
			return ;
		}
			
		std::vector<std::vector<int> > bucket(10);
		std::for_each(v.begin(),v.end(),[&](int num)
		{
			bucket[getDim(num,digit)].push_back(num);
		});

		for(int i=0;i<10;i++)
		{
			bucketSortRec(bucket[i],digit-1,sorted);
		}
	}

	void bucketSort(std::vector<int> &v, int digit, std::vector<int> &sorted)
	{
		std::vector<std::vector<int> > bucket(10);
		std::for_each(v.begin(),v.end(),[&](int num)
		{
			bucket[getDim(num,digit)].push_back(num);
		});

		std::for_each( bucket.begin(), bucket.end(), [&]( std::vector<int> &sub )
		{
			std::sort( sub.begin(), sub.end());
			sorted.insert(sorted.end(),sub.begin(),sub.end());
		});
	}

	void radixSort(std::vector<int> &v, std::vector<int> &sorted, std::string &method)
	{
		int max = -1,digit = 0;

		for(std::vector<int>::iterator iter=v.begin();iter!=v.end();iter++)
		{
			max = max >= *iter ? max : *iter;
		}
		while(max / 10)
		{
			max /= 10;
			digit++;
		}

		if(!method.compare("lsd"))
		{
			std::vector<int> tmpArray(10);
			std::vector<int> tmp(v.size(),0);
			sorted = std::vector<int>(v);
			int length = v.size();
			int log = 1;

			for(int j=0;j<=digit;j++)
			{
				for(std::vector<int>::iterator iter=tmpArray.begin();iter!=tmpArray.end();iter++)
				{
					*iter = 0;
				}

				for(int i=0;i<length;i++)
					tmpArray[(sorted[i]/log)%10]++;
				for(int i=1;i<10;i++)
					tmpArray[i] += tmpArray[i-1];

				for(int i=length-1;i>=0;i--)
				{
					tmp[tmpArray[(sorted[i]/log)%10] - 1] = sorted[i];
					tmpArray[(sorted[i]/log)%10]--;
				}

				for(int i=0;i<length;i++)
					sorted[i] = tmp[i];
				log *= 10;
			}
		}
		else if(!method.compare("msd"))
		{
			bucketSortRec(v, digit, sorted);
		}
		else
		{
			std::cerr << "Wrong method!" << std::endl;
		}
	}
}

/* main.cpp */
#include "linearsort.h"

using namespace std;

int main()
{
	vector<int> v, sorted;
	
	for(int i=0;i<100;i++)
	{
		v.push_back(rand() % 100);
	}
	algo::countingSort(v,100,sorted);
	cout << "计数排序：" << endl;
	copy(sorted.begin(),sorted.end(),std::ostream_iterator<int>(std::cout, "\t"));
	v.clear();
	sorted.clear();
	getchar();

	for(int i=0;i<100;i++)
	{
		v.push_back(rand() % 1000);
	}
	algo::bucketSort(v,2,sorted);
	cout << "桶排序：" << endl;
	copy(sorted.begin(),sorted.end(),std::ostream_iterator<int>(std::cout, "\t"));
	v.clear();
	sorted.clear();
	getchar();

	for(int i=0;i<100;i++)
	{
		v.push_back(rand() % 1000);
	}
	algo::radixSort(v,sorted,string("lsd"));
	cout << "基数排序LSD：" << endl;
	copy(sorted.begin(),sorted.end(),std::ostream_iterator<int>(std::cout, "\t"));
	v.clear();
	sorted.clear();
	getchar();

	for(int i=0;i<100;i++)
	{
		v.push_back(rand() % 1000);
	}
	algo::radixSort(v,sorted,string("msd"));
	cout << "基数排序MSD：" << endl;
	copy(sorted.begin(),sorted.end(),std::ostream_iterator<int>(std::cout, "\t"));
	v.clear();
	sorted.clear();

	return 0;
}
```

测试结果

![](/images/algorithm/linearsort-result1.jpg)

![](/images/algorithm/linearsort-result2.jpg)