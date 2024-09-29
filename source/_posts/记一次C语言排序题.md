---
title: 记一次C语言排序题
date: 2024-09-09 10:06:07
tag: C语言
description: 偶然接触到的算法题
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/cLanage.png
---

# [COCI2006-2007#2] ABC

## 题面翻译

**【题目描述】**

三个整数分别为 $A,B,C$。这三个数字不会按照这样的顺序给你，但它们始终满足条件：$A < B < C$。为了看起来更加简洁明了，我们希望你可以按照给定的顺序重新排列它们。

**【输入格式】**

第一行包含三个正整数 $A,B,C$，不一定是按这个顺序。这三个数字都小于或等于 $100$。第二行包含三个大写字母 $A$、$B$ 和 $C$（它们之间**没有**空格）表示所需的顺序。

**【输出格式】**

在一行中输出 $A$，$B$ 和 $C$，用一个 ` `（空格）隔开。 

感谢 @smartzzh 提供的翻译

## 样例 #1

### 样例输入 #1

```
1 5 3
ABC
```

### 样例输出 #1

```
1 3 5
```

## 样例 #2

### 样例输入 #2

```
6 4 2
CAB
```

### 样例输出 #2

```
6 2 4
```

```C
#include <stdio.h>

// 递归函数，用于将数组从大到小
void recursiveSort(int num[], int start, int size) {
    int i, minIndex, temp;  // 将所有变量提前声明
	
    if (start >= size - 1) {
        // 递归基准情况：当 start 达到数组的最后位置时，排序完成
        return;
    }
    // 找到从 start 到 size - 1 范围内的最小值
    minIndex = start;
    for (i = start + 1; i < size; i++) {  // 将 i 的声明移到循环外
        if (num[i] < num[minIndex]) {
            minIndex = i;
        }
    }
    // 将最小值与 start 位置的元素交换
    if (minIndex != start) {
        temp = num[start];
        num[start] = num[minIndex];
        num[minIndex] = temp;
    }
    // 递归调用，对剩下的部分进行排序
    recursiveSort(num, start + 1, size);
}

int main() 
{
    int num[3],var;  // 初始化数组
	char order[4];
	printf("输入三个数字,以空格区分\t[例:1 2 3]\n");
	scanf("%d%d%d",&num[0],&num[1],&num[2]);
	printf("输入三个字母，例: ABC\t[例:1 2 3]\n");
	scanf("%s", order);
    // 调用递归排序函数，对 num 进行排序
    recursiveSort(num, 0, 3);
	//根据输入的字母顺序重新对数字数组排序
	for(var = 0;var<3;var++)
	{
		if(order[var] == 'A' || order[var] == 'a')
		{
			//说明第一个要输出最小值
			printf("%d\t", num[0]);
		}
		else if(order[var] == 'B' || order[var] == 'b')
		{
			//输出第二小的值
			printf("%d\t", num[1]);
		}
		else if(order[var] == 'C' || order[var] == 'c')
		{
			//输出最大的数值
			printf(%d\t", num[2]);
		}

	}
    return 0;
}
```

## 思路解析

1、利用递归函数对我们数字数组进行排序

2、遍历我们输入的字符数组，分别判断A、B、C三种情况

3、因为提前排序过了，所以只需要用if..else分别判断输出最小值、中间值、最大值

注：之所以用两个else..if，是因为预防输入不是ABC内的情况，如ABV

这个时间太长了


```c
#include <stdio.h>

// 选择排序函数，用于对数组进行升序排序
void iterativeSort(int num[], int size) 
{
    int i, j, minIndex, temp;   
    // 外层循环遍历数组的每个元素
    for (i = 0; i < size - 1; i++) 
    {
        minIndex = i;  // 假设当前元素是最小的      
        // 内层循环查找剩余未排序部分中的最小元素
        for (j = i + 1; j < size; j++) 
        {
            if (num[j] < num[minIndex]) 
            {
                minIndex = j;  // 更新最小元素的索引
            }
        }
        // 如果找到比当前元素更小的元素，则交换它们
        if (minIndex != i) 
        {
            temp = num[i];
            num[i] = num[minIndex];
            num[minIndex] = temp;
        }
    }
}
int main() 
{
    int num[3];  // 存储输入的三个数字
    int var;     // 用于遍历的变量
    char order[4];  // 存储用户输入的顺序，长度为4以容纳结束符
    // 输入三个数字，空格分隔
    scanf("%d %d %d", &num[0], &num[1], &num[2]);
    // 输入顺序字符，例如 "ABC"
    scanf("%s", order);
    // 对数字进行排序（升序）
    iterativeSort(num, 3);
    // 按用户输入的顺序输出排序后的数字
    for (var = 0; var < 3; var++) 
    {
        // 根据顺序字符输出对应的数字
        if (order[var] == 'A') 
        {
            printf("%d", num[0]);  // 输出最小的数字
        } 
        else if (order[var] == 'B') 
        {
            printf("%d", num[1]);  // 输出中间的数字
        } 
        else if (order[var] == 'C') 
        {
            printf("%d", num[2]);  // 输出最大的数字
        }
        // 如果不是最后一个数字，则输出空格
        if (var < 2) 
        {
            printf(" ");
        }
    }
    return 0;
}
```
虽然也没节省多少时间，因为之前不通过主要是因为结果集没有按空格区分，删除掉注释和提示信息（因为不用人工输入数据），缩减了代码量

但是看了一圈发现这个代码还是太臃肿了