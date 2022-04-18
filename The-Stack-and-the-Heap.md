# The Stack and the Heap

> 用 Go 来实现栈与堆数据结构



Yes, I realise that they are quite different — stack is linear while heap is hierarchical. However when we’re using a heap as a priority queue, the operations are almost the same, though what we get from these data structures are quite different.


栈与堆是很不同的数据结构，栈是一种线性数据结构，堆是分层的数据结构。
然而当我们把堆当做优先队列来使用，两者的操作是是相同的，尽管我们从这些数据结构中得到是完全不同的。

让我们来简要介绍一下。


## Stack

A stack is one of the easiest data structure around. Imagine a stack of pebbles (that’s pretty much how the name comes about). The pebbles don’t need to be the same size, but when we add a pebble to the stack we add to the top of the stack. When we want to take a pebble from the stack, we take from the top again.

栈是我们所遇到中的最容易的数据结构之一。

想象层层叠叠的鹅卵石，其大小不一，我们通过向顶部来添加一个鹅卵石，我们想要从这一堆拿走一个时，也是从顶部。

In other words, the order of how we add or take from the stack is Last-In-First-Out (LIFO). The operations to add or take from the stack are called push and pop respectively.

换句话来说，我们从栈中添加与移除的顺序是 Last-In-First-Out(LIFO) 先进后出的。从栈中添加与移除的操作各自称为 push 与 pop.

Let’s see how to implement a stack in Go.

让我们来看下如果在 Go 中实现栈数据结构。


```golang
type Stack struct {
	elements []string
	mutex    sync.RWMutex
}
```


It’s simply struct that wraps around a slice and if we want to make it concurrency-safe, we throw in a mutex. The push and pop functions are quite straightforward.

Stack 只是一个包含切片的结构体，并且如果我们想要其实现并发安全，我们向其内部包含一个 mutex 锁。

push 与 pop 的方法都十分直接简单。

```golang
func (s *Stack) Push(el string) {
	s.mutex.Lock()
	defer s.mutex.Unlock()
	s.elements = append(s.elements, el)
}

func (s *Stack) Pop() (el string, err error) {
	if len(s.elements) == 0 {
		err = errors.New("empty stack")
		return
	}
	s.mutex.Lock()
	defer s.mutex.Unlock()
	el = s.elements[len(s.elements)-1]
	s.elements = s.elements[:len(s.elements)-1]
	return
}
```

Pushing on to the stack is just appending to the slice, and popping from the stack simply removes the last element from the slice.

入栈操作只是往切片进行追加，出栈操作只是简单地从切片移除最后一个元素。

A quick aside on the mutex. I used the sync.RWMutex here because I can do write locks if I’m modifying the stack, but if I just want to read the stack, I can do read locks. Both push and pop functions shown here are writes so I’m using the write locks, but if I implement a peek or a size function, I will only use read locks.

快速说明一下互斥锁。

在这里我使用了 sync.RWMutes 读写互斥锁，因为当我修改栈时我可以加写锁，但是当我只是想要读取栈数据时，我可以做读锁。

push 与 pop 演示使用写锁，不过当我实现 peek 与 size 方法时，我会只用到读锁。


## Heap

Heap
Now, a heap is a different thing altogether. Yes, I know I said they are very similar but internally, they are very different. A stack is like a stack of pebbles, but a heap is a, well, heap of pebbles. A rock cairn, if you will.

现在开始，堆是完全不同的东西

是的，我知道我说过它们很相似，但是在内部，它们是非常不同的。

栈好似层层叠叠的鹅卵石，但是堆，一堆鹅卵石，如果你愿意可以说是石堆。

A heap is a tree-like data structure that satisfies the heap property. There are 2 types of heaps, the first is the min heap where the heap property is such that the key of the parent node is always smaller than that of the child nodes. The second is the max heap where the heap property, as you would expect it, is such that the key of the parent node is always larger than that of the child nodes. A popular heap implementation is the binary heap, where each node has at most 2 child nodes.

堆是满足堆属性的树状数据结构

有两种堆类型：
- 第一种是最小堆，其堆属性使得父节点的键总是小于子节点的键。
- 第二种是最大堆，正如你所预料，其堆属性使得父节点的键问题大于子节点的键。


一种受欢迎的堆实现是二叉堆，每个节点都最多拥有 2 个子节点。


图


Let’s see how a binary heap can be implemented in Go. You would be surprised that heaps are commonly implemented using the exactly the same way a stack is implemented, using an array (in our case, a slice)!


让我们来看下如果在 Go 中实现二叉堆。

你会惊讶堆的普遍实现使用了栈实现的相同方式 - 使用数组（这儿是切片）。

```golang
type Heap struct {
	elements []int
	mutex    sync.RWMutex
}
```

This is how the max heap in the figure above would look like, laid out in the form of an array.


