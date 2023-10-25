# 函数参数值传递

[参考：Draveness博客](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/)

* slice作为参数
```go
func TestPtr(t *testing.T) {
	a := []int{1, 2, 3}
	m := map[int]int{1: 1, 2: 2, 3: 3}
	t.Log(unsafe.Sizeof(a), unsafe.Sizeof(m))  // 24 8
	t.Log(unsafe.Pointer(&a), unsafe.Pointer(&m)) // 0xc000134c78 0xc0001287d0

	f := func(sl []int, mp map[int]int) {
		t.Log(unsafe.Pointer(&sl), unsafe.Pointer(&mp)) // 0xc000134c90 0xc0001287d8
		sl[0] = 0
		sl = append(sl, 4) // 加不到a上

		m[1] = 0
		m[4] = 4
	}
	f(a, m)
	t.Log(a) // [0 2 3]
	t.Log(m) // map[1:0 2:2 3:3 4:4]
}
```

疑惑：
1. 为啥修改可以应用到原切片但是不能append？
2. map为啥在函数内部发生扩容，这个动作也会应用到外部的map中?

原因：

* 参数值传递
* slice底层结构
* map变量实质

## 参数传递

* 传递基本类型时，会拷贝其值；
* 传递切片或数组时，会拷贝切片或数组中的所有值；
* 传递结构体时：会拷贝结构体中的全部内容；
* 传递结构体指针时：会拷贝结构体指针的值；

## slice数据结构

```go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```
* Data 是指向数组的指针;
* Len 是当前切片的长度；
* Cap 是当前切片的容量，即 Data 数组的大小：


## 原因
切片在函数参数中进行传递，相当于传递SliceHeader结构体，会拷贝结构体中的全部内容。

根据索引进行修改时，会根据Data字段找到底层的数组并修改对应的值；由于函数内的切片sl和函数外的切片a虽然不是同一个SliceHeader结构体，但是**其中的全部内容都相同**，对data指向的底层数据进行修改会在两个结构体之间**共享**。

而append不会影响函数外切片a的原因是，切片的数据是根据Data指向的首地址和Len圈定的范围共同决定的，无论函数内部如何append，外边原切片的**Len是不变**的，所以确定的底层数组长度也是不变的，添加到数组末尾的元素不在Len圈定的范围之内。

## map变量实质

因为map、channel这两种类型的值其实是指向runtime.hmap与runtime.hchan的**指针**，而slice类型就是runtime.sliceHeader类型。

根据上边代码中这一行输出可以看出：

```go
t.Log(unsafe.Sizeof(a), unsafe.Sizeof(m))  // 24 8
```

输出中24等于SliceHeader中三个字段大小之和，而8则对应着一个 `uintptr` 的大小

所以slice和map作为形参进行传递分别对应着上文提到的参数传递规则：

* 传递结构体时：会拷贝结构体中的全部内容；  // slice
* 传递结构体指针时：会拷贝结构体指针的值；  // map