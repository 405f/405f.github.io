---
title: 堆排序
date: 2020-06-02 21:25:54
tags: [redis]
---



```go

package heap

import "fmt"

func swap(A []int, i, j int) {
	A[i], A[j] = A[j], A[i]
}

//// heapify 最大堆的性质, 建立一个最大堆
//// A[left] <A[i] && A[right] < A[i]
//通过下标进行最大值比较过程
func heapify(arr []int, heapSize, i int) {
	left := 2*i + 1
	right := 2*i + 2
	largest := i
	if left < heapSize && arr[left] > arr[largest] {
		largest = left
	}
	if right < heapSize && arr[right] > arr[largest] {
		largest = right
	}
	if largest != i {
		arr[largest], arr[i] = arr[i], arr[largest]
		heapify(arr, largest, heapSize)
	}
}

func heapSort(A []int) {
	n := len(A)
	for i := n / 2; i >= 0; i-- { // 构建最大堆
		heapify(A, n, i)
	}
	for i := n - 1; i > 0; i-- {
		// Move current root to end
		swap(A, 0, i) // 把最大的数据提取出来
		heapify(A, i, 0)
	}
	fmt.Println("sort", A)
}


```


## container heap 实现

```go

func Init(h Interface) {
	// heapify
	n := h.Len()
	for i := n/2 - 1; i >= 0; i-- {
		down(h, i, n)  //是从最后一颗树开始的
	}
}


```

1. down 的具体实现

```go

// n 代表这个数组的长度,构建最小堆
func down(h Interface, i0, n int) bool {
	i := i0
	for {
		j1 := 2*i + 1 // 左节点
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow 检查边界条件 ，确保left index 不超过数组的长度
			break
		}
		j := j1 // 左节点
		if j2 := j1 + 1; j2 < n && h.Less(j2, j1) { // j2=右节点， 右节点< 左节点
			j = j2 // = 2*i + 2  // right child
		}
		if !h.Less(j, i) {  // 右节点 == 当前节点的值
			break
		}
		h.Swap(i, j) //  右节点 > 当前节点的值, 交换
		i = j
	}
	return i > i0
}

```