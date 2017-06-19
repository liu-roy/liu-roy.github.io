---
title: 简单排序算法实现
date: 2016-01-18 23:03:50
categories: 算法
tags: [排序]
---

# 简单排序算法实现

闲来无事，简单回顾了一下排序算法
------
>* 简单插入排序,哈希排序
>* 冒泡排序,快速排序
>* 简单选择排序,堆排序

<!--more-->
```c++
#include <stdio.h>
#include<stdlib.h>

void insertSortForward(int orig[], int size);
void insertSortBackward(int orig[], int size);
void bubbleSort(int orig[], int size);
void selectSort(int orig[], int size);
void shellSort(int orig[], int size);
void HeapSort(int orig[], int size);
void QuickSort(int[], int, int);
void swap(int *, int *);
int Partition(int orig[], int left, int right);
void BuildHeap(int orig[],int size);
void HeapAdjust(int[], int, int);

int main() {
	int a[] = { 1,3,2,4,5,7,0,8,8 };
	//bubbleSort(a, 7);
	//insertSortForward(a, 8);
	//insertSortBackward(a, 8);
	//selectSort(a, 8);
	//shellSort(a, 8);
	//HeapSort(a, 9);
	QuickSort(a, 0, sizeof(a)/sizeof(a[0])-1);
	for (int i = 0; i < sizeof(a) / sizeof(a[0]); i++)
		printf("%d ", a[i]);
	printf("\n");
	system("pause");
	return 0;
}

void shellSort(int orig [], int size) //Shell排序
{
	int i, j, k, t;
	k = (size / 2) % 2 == 0 ? size / 2 + 1 : size / 2; 
	while (k > 0)
	{
		for (int j = k; j < size; j++) {
			t = orig[j];
			i = j - k;
			while (i >= 0 && t < orig[i]) {
				orig[i+k] = orig[i];
				orig[i] = t;
				i = i - k;
			}
		}
		k /= 2;
		if (k==0)
		{
			break;

		}
	}
	for (int i = 0; i < size; i++)
		printf("%d ", orig[i]);
	printf("\n");
		
}
void HeapSort(int orig[], int size)
{
	BuildHeap(orig, size);
	for (int i = 0; i < size; i++)
		printf("%d ", orig[i]);
	printf("\n");
	while(size > 0)
	{
		printf("%d ", orig[0]);
		orig[0] = orig[ --size];
		HeapAdjust(orig, 0,size);
	}
}
void QuickSort(int orig[], int left, int right)
{
	if (left < right) {

		int point = Partition(orig, left, right);
		QuickSort(orig, left, point - 1);
		QuickSort(orig, point + 1, right);
	}
}
void swap(int *a, int*b) {
	int temp;
	temp = *a;
	*a = *b;
	*b = temp;
}
int Partition(int orig[], int left, int right)
{
	int prikey = orig[right]; 
	while (left<right)
	{
		while (left<right&&prikey >= orig[left]) left++;
		swap(&orig[left], &orig[right]);
		while (left<right&& prikey <= orig[right]) right--;
		swap(&orig[left], &orig[right]);
	}
	return left;
}
void BuildHeap(int orig[], int size)
{
	int k;
	for (k=size/2-1; k>=0; k--)
	{
		HeapAdjust(orig, k, size);
	}
	

}
void HeapAdjust(int orig[], int adjustPort, int size) {
	
	int min;   //记录左右节点中最小的
	int nextPoint; // 下一步
	while (adjustPort*2+1<size-1) // 还有孩子
	{	
		int leftChild = adjustPort * 2+1;
		int rightChild = adjustPort * 2 + 2;
		min = orig[leftChild];
		nextPoint = leftChild;
		if (rightChild<=size-1&&orig[rightChild]<orig[leftChild])
		{
			min = orig[rightChild];
			nextPoint = rightChild;
		}
		if (orig[adjustPort]>min)
		{
			orig[nextPoint] = orig[adjustPort];
			orig[adjustPort] = min;
			adjustPort = nextPoint;
		}
		else
		{
			break;
		}
	}
}
void insertSortForward(int orig[], int size)
{
	int i, j;
	for (i = 1; i < size; i++) {
		int temp = orig[i];
		int index = i;
		j = i - 1;
		while (j >= 0 && orig[j] > temp)
		{
			orig[index] = orig[j];
			index = j;
			j--;
		}
		orig[j + 1] = temp;
	}
	for (int i = 0; i < size; i++)
		printf("%d ", orig[i]);
	printf("\n");
}

void insertSortBackward(int orig[], int size) // 从后往前
{
	int i, j;
	int left, right;
	for (i = 1; i < size; i++) {
		for (j = 0; j < i; j++) {
			if (orig[i] <= orig[j])
			{
				left = orig[i];
				right = i;
				while (i - j)
				{
					orig[i] = orig[i - 1];
					i--;
				}
				i = right;
				orig[j] = left;
			}
		}
		
	}
	for (int i = 0; i < size; i++)
		printf("%d ", orig[i]);
	printf("\n");
}

void bubbleSort(int orig[], int size) {
	int i = 0;
	int j = 0;
	for (i = 0; i < size; i++) {
		for (j = 0; j < size - i; j++) {
			if (orig[j] > orig[j + 1]) {
				int temp = 0;
				temp = orig[j];
				orig[j] = orig[j + 1];
				orig[j + 1] = temp;
			}
		}
	}
	for (int i = 0; i < size; i++)
		printf("%d ", orig[i]);
	printf("\n");
}

void selectSort(int orig[], int size)
{
	for (int i = 0; i < size; i++) {
		int min = orig[i];
		int index = i;
		for (int j = i+1; j < size; j++)
		{
			if (min > orig[j]) {
				min = orig[j];
				index = j;
			}
		}
		orig[index] = orig[i];
		orig[i] = min;
	}
	for (int i = 0; i < size; i++)
		printf("%d ", orig[i]);
	printf("\n");
}
```