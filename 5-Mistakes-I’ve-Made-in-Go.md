

## 5 Mistakes I’ve Made in Go


>犯错是人性，宽恕是神圣的。
— 亚历山大·波普



我在写 go 时犯过一些错误,尽管这些错误可能不会造成任何各类的 error,但是会潜在地影响软件.


### Inside Loop

内循环


有几种方式造成内循环出现混乱,这是需要你注意到的.



#### Using reference to loop iterator variable

基于效率的考量,循环迭代器的变量在每次循环中采用的是单个变量的不同值.

这可能导致不易察觉的行为出现

```golang
	old := []int{1, 2, 3}

	// int pointer slice
	newS := make([]*int, 0, 3)

	// loop copy
	for _, num := range old {
		// get reference
		newS = append(newS, &num)
	}

	fmt.Println("Values:", *newS[0], *newS[1], *newS[2])
	fmt.Println("Addresses:", newS[0], newS[1], newS[2])
```
output:
```
Values: 3 3 3
Addresses: 0xc00010c000 0xc00010c000 0xc00010c000
```

正如你所看见的, `newS` 切片中所有的元素都是相同的.

事实上这也容易解释:每次迭代我们追加 `num` 的地址到 `newS` 切片,在上面已经提到过 `num` 在迭代中是单个变量的不同值.
最终 `num` 指向的是 `old` 切片的最后一个值的地址.


这个问题可以通过创建一个新的变量来保存迭代数量解决:
```golang
... ...

// loop copy
	for _, num := range old {
		// get reference
		copyNum := num
		newS = append(newS, &copyNum)
	}


... ...
```
output
```
Values: 1 2 3
Addresses: 0xc00001c0b0 0xc00001c0b8 0xc00001c0c0
```


#### 1.2 Calling WaitGroup.Wait in loop





















