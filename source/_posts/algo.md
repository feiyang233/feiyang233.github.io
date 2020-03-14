---
title: Algorithm
date: 2019-12-01 23:07:37
tags: go
categories: algorithm
---
Record the learning algorithm experience.
<!--more-->
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191227141136.jpeg)

# reference
1. [背包问题](https://mp.weixin.qq.com/s/FQ0LCROtEQu3iBZiJb0VBw)

# Math
1. Reverse Integer

```go
func reverse(x int) int {
    //math.MinInt32 = -2147483648
    //math.MaxInt32 = 2147483647
    var result int
    for x!=0{
        result=result*10+x%10
        if result > 2147483647  || result < -2147483648{
            return 0
        }
        x/=10
    }
    return result
}
```
# quickSort
1. Based on recursive

```go
package main

import "fmt"

func quickSort(arr []int, startIndex, endIndex int )  {

	if startIndex >= endIndex {
		return
	}
	pivotIndex:= partition(arr,startIndex,endIndex)
	quickSort(arr,startIndex,pivotIndex-1)
	quickSort(arr,pivotIndex+1,endIndex)
}

func partition(arr []int, startIndex, endIndex int )  int {

	pivot:=arr[startIndex]
	mark:=startIndex

	for i:=startIndex+1; i<= endIndex; i++{
		if arr[i]<pivot {
			mark++    //  numbers that smaller pivot ++
			arr[i],arr[mark] = arr[mark], arr[i]
		}
	}
	arr[startIndex] = arr [mark]
	arr[mark] = pivot
	return mark
}

func main() {

	var arr = []int{4,4,6,5,3,2,8,1}
	quickSort(arr,0, len(arr)-1)
	fmt.Print(arr)
}


```

2. Use stack

```go
package main

import "fmt"
import "github.com/golang-collections/collections/stack"

func quickSort(arr []int, startIndex, endIndex int )  {

	quickSortStack := stack.New()
	rootParam := make(map[string]int)
	rootParam["startIndex"] = startIndex
	rootParam["endIndex"] = endIndex
	quickSortStack.Push(rootParam)

	for quickSortStack.Len() > 0 {

		param := quickSortStack.Pop().(map[string]int)
		pivotIndex := partition(arr, param["startIndex"], param["endIndex"] )

		if param["startIndex"] < pivotIndex-1 {
			leftParam := make(map[string]int)
			leftParam["startIndex"] = param["startIndex"]
			leftParam["endIndex"] = pivotIndex-1
			quickSortStack.Push(leftParam)
		}
		if pivotIndex+1 < param["endIndex"] {

			rightParam := make(map[string]int)
			rightParam["startIndex"] = pivotIndex+1
			rightParam["endIndex"] = param["endIndex"]
			quickSortStack.Push(rightParam)
		}
	}
}

func partition(arr []int, startIndex, endIndex int )  int {

	pivot:=arr[startIndex]
	mark:=startIndex

	for i:=startIndex+1; i<= endIndex; i++{
		if arr[i]<pivot {
			mark++    //  smaller pivot numbers ++
			arr[i],arr[mark] = arr[mark], arr[i]
		}
	}
	arr[startIndex] = arr [mark]
	arr[mark] = pivot
	return mark
}

func main() {

	var arr = []int{4,4,6,5,3,2,8,1}
	quickSort(arr,0, len(arr)-1)
	fmt.Print(arr)
}


```

# Hash Table
1. Two Sum
```go
func twoSum(nums []int, target int) []int {
    
    dic := make(map[int]int)
    for i, v := range nums {
        
        k,ok := dic[target-v]
        
        if ok {
            return []int{k, i}
        }
        dic[v]=i     
    }
    return nil  
}

```
2. least recently used
