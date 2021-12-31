

# Explain How to delete an element from a Slice in Golang




## Method 1

```golang
func remove(slice []int, removeIndex int) []int {
	return append(slice[:removeIndex], slice[removeIndex+1:]...)
}
```


使用 re-slicing 方式来跳过想要删除元素
slice[i:j] 包含 i 下标不包含 j 下标 

使用 append 来追加另外一个切片, 通过 ... 语法来对切片进行展开.
append(slice, elem ...) = append(slice, slice...)

根据切片共享储存, slice[:removeIndex] 与原来的 slice 共享相同的内存区域,所以切片的容量 capacity 都为 cap(slice).
删除一个元素对原来的容量不会触发扩容操作,不存在扩容开销.

performance 消耗在于删除元素索引后的元素需要一个个向前移动,时间复杂度为 O(n).




## Method 2

```golang
func remove(s []int, i int) []int {
    s[i] = s[len(s)-1]
    return s[:len(s)-1]
}
```

如果对列表的顺序没有要求, 可以通过将欲删除的元素保存为最后一个元素值.再通过 re-slicing 的方式达到删除目的,这样时间复杂度为 O(1).


## 参考

- [1] [how-to-delete-an-element-from-a-slice-in-golang](https://stackoverflow.com/questions/37334119/how-to-delete-an-element-from-a-slice-in-golang)
- [2] [A closer look at Go's append function](https://dev.to/andyhaskell/a-closer-look-at-go-s-slice-append-function-3bhb)























