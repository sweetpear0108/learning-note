# map

# 参考
[Draveness博客](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)

## 数据结构
runtime/map.go
```go
// A header for a Go map.
type hmap struct {
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8  // 枚举值: 1 2 4 8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

// runtime.map中定义的结构 A bucket for a Go map.
type bmap struct {
	tophash [bucketCnt]uint8  // 存储了键的哈希的高 8 位
}

// 运行期间重构的bmap结构
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```
![mapstructure](https://img.draveness.me/2020-10-18-16030322432679/hmap-and-buckets.png)

## 初始化

* 根据元素数量和size计算出需要的最小桶数量，条件：低于装载因子（```装载因子:=元素数量÷桶数量```）

* 桶的数量多于 2<sup>4</sup> 时，会额外创建 2<sup>B-4</sup> 个溢出桶

* 分配内存空间，正常桶和溢出桶在内存中的存储空间是连续的（在初始化时，后续可能因为扩容而不连续），只是被 `hmap` 中的不同字段引用（`hmap.buckets`, `hmap.extra.overflow`）

## 读写操作
### 访问

![mapaccess](https://img.draveness.me/2020-10-18-16030322432560/hashmap-mapaccess.png)

如果在扩容过程中（ `h.oldbuckets` 不为空）并且桶中数据没有被分流（ `evacuated` ），则下述步骤1.2.3则在 `h.oldbuckets`， `h.extra.oldoverflow` 数组中寻找。

1. 根据 `hmap.B` bucket的次方数（2^B）计算掩码（比如B=3，则掩码为……0111），通过&运算计算出key**哈希值**的低**B（不是8是B）**位，得到对应桶在 `hmap.buckets` 数组中的索引值，从而可以通过数组首地址和索引值和元素类型三者计算出所求bucket元素（ `bmap` ）内存地址；

2. 拿到相应的bucket后，依次遍历**正常桶**和**溢出桶**中的 `bmap.tophash` ，先比较key的高8位和 `tophash[i]` ，后比较传入的和桶中的值以**加速数据的读写**（tophash的作用，类似于**索引**）。

3. 找到key后，再用 `bmap` 内存地址加上偏移量找到对应的value值

### 写入

1. 首先根据 `hmap.flags` 判断是否有并发写入，有则panic；并将flags置为写入状态；

2. 如果map在扩容过程中（ `h.oldbuckets` 不为空），则需要先进行增量分流（将旧桶分流（如果没被分流过）并额外分流 `hmap.nevacuate` 对应的桶，即第一个还没被分流的桶）。

3. 接下来主要逻辑和访问相似，依次遍历正常桶和溢出桶：

* 如果遇到相同的 `tophash[i]` ，则向下寻找对应的key，若key值相同则**返回对应的value内存地址**。如果不同则跳过（这里也说明tophash数组是可能会有重复值的，但是**不会**大量重复，因为在选择bukect时是按照哈希低B位分桶的，减少同一个桶中有大量相等 `tophash` 的概率影响性能。）；

* 如果遇到空的tophash[i]，则获取到key和待赋值的value以及tophash[i]的内存地址，并在逻辑中标记状态为已填入，避免重复写入；

* 如果所有桶中都没有空位，则创建一个新的溢出桶作为接收的桶；

4. 在函数最后最后**返回value内存地址**。而真正的赋值操作是在**编译**期间插入的。

### 扩容

#### 什么时候需要扩容

1. 通过 `hmap.oldbuckets` 是否为空判断map是否在扩容过程中；（map扩容不是一个原子过程）

2. 满足以下两种情况之一

* 装载因子已经超过6.5；   ->  翻倍扩容

* 哈希使用了太多溢出桶；   ->  等量扩容 

问：为什么会等量扩容？

答：持续向哈希中插入数据并将它们全部删除时，如果哈希表中的数据量没有超过阈值，就会不断积累溢出桶造成缓慢的内存泄漏。

#### 扩容步骤

![evacuate](https://img.draveness.me/2020-10-18-16030322432579/hashmap-evacuate-destination.png)

1. 空间分配：在**写入**操作中（步骤2.3之间），如果碰到需要扩容的情况，则将 `hmap.buckets`, `hmap.extra.overflow` 中的数据移到 `h.oldbuckets`， `h.extra.oldoverflow` 中，
并创建新的正常桶和溢出桶数组（等量或翻倍，因扩容类型不同而异）。这一步骤只是**初始化**了相应数组，完成初始化后会返回到当前写入操作函数开头的分流函数位置（[写入步骤2](#写入)）；

2. 增量分流：
	1. 声明 `evacDst` 结构体

	* 对于**翻倍扩容**，声明两个 `evacDst` 结构体保存分配上下文并指向新桶（其实就是保存了下一个分流出的元素应该插入的位置（`evacuation destination`），包括键/值地址、索引值、新桶地址）；

	* 对于**等量扩容**，声明一个 `evacDst` 结构体；

	2. 遍历正常桶和溢出桶中所有的元素，分流到 `evacDst` 指向的位置，如果分流桶装满了就新建一个溢出桶；

	3. 如果 `h.nevacurate` 等于当前桶序号，说明当前序号之前的桶已经分流完了，则更新分流进度（ `h.nevacurate++` 直到碰到一个没被分流的旧桶）；

	4. 如果 `h.nevacurate` 等于旧桶数量，说明map中所有桶都分流完了，所以将 `h.oldbuckets`， `h.extra.oldoverflow` 置空释放内存；

### 删除

主要逻辑和写入基本相同，遍历找到键/值地址，释放内存，更改tophash[i]的状态。