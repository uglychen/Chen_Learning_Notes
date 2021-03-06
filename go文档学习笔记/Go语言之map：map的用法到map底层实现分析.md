**带着几个问题阅读本文**：  
**1. go map 实现方法？如何解决hash冲突的？**    
**2. go map是否线程安全？**  
**3. go map 的扩容机制？**


# 什么是map？
- [ ] 由一组 <key, value> 对组成的抽象数据结构，并且同一个 key 在map中只会出现一次

map 的设计也被称为 “The dictionary problem”，它的任务是设计一种数据结构用来维护一个集合的数据，并且可以同时对集合进行增删查改的操作。最主要的数据结构有两种：哈希查找表（Hash table）、搜索树（Search tree）。

哈希查找表用一个哈希函数将 key 分配到不同的桶（bucket，也就是数组的不同 index）。这样，开销主要在哈希函数的计算以及数组的常数访问时间。在很多场景下，哈希查找表的性能很高。

哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：链表法和开放地址法。链表法将一个 bucket 实现成一个链表，落在同一个 bucket 中的 key 都会插入这个链表。开放地址法则是碰撞发生后，通过一定的规律，在数组的后面挑选“空位”，用来放置新的 key。

搜索树法一般采用自平衡搜索树，包括：AVL 树，红黑树 c++中STL_MAP 是红黑树结构

自平衡搜索树法的最差搜索效率是 O(logN)，而哈希查找表最差是 O(N)。当然，哈希查找表的平均查找效率是 O(1)，如果哈希函数设计的很好，最坏的情况基本不会出现。还有一点，遍历自平衡搜索树，返回的 key 序列，一般会按照从小到大的顺序；而哈希查找表则是乱序的

# map的用法

```
package main

import "fmt"

func main(){
	m1 := map[string]string{ // :=创建
		"name": "小明",
		"age":  "20",
	}
    //遍历map
	for k ,v :=range m1{
		fmt.Println(k, v)
	}

	// 测试key是否存在，存在ok=true 否则ok=false
	if name, ok := m1["name"]; ok { //如果name存在ok就为true
		fmt.Println(name, ok)
	}

	m2 := make(map[string]string) //通过make创建
	m2["city"]  = "shanghai"

	//修改
	m2["city"]  = "beijing"

	//删除key
	delete(m2, "city")

	var m3 map[string]int //通过var 注意此时的map是一个nil map 无法插入key/value
	fmt.Println(m3)
	m3 = make(map[string]int)
	m3["count"] = 100
}
```

# map的类型:

```
golang中的map是一个 指针。当执行语句 make(map[string]string) 的时候，其实是调用了 makemap 函数：

// file: runtime/hashmap.go:L222
func makemap(t *maptype, hint64, h *hmap, bucket unsafe.Pointer) *hmap
显然，makemap 返回的是指针。
```
**因为返回的是指针，map作为参数的时候，函数内部能修改map。**

**我们知道slice 也可以使用make初始化，makeslice返回的是结构体,slice作为参数的时候,函数内部修改可能会影响slice，这涉及到slice的具体实现，这部分内容下篇文章仔细研究。**

```
func makeslice(et *_type, len, cap int) slice

// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int // 长度 
    cap   int // 容量
}
```


# go hmap 数据结构
go map 采用的是哈希查找表，并且使用链表解决哈希冲突

```
type hmap struct {
    count     int //map元素的个数，调用len()直接返回此值
    
    // map标记:
    // 1. key和value是否包指针
    // 2. 是否正在扩容
    // 3. 是否是同样大小的扩容
    // 4. 是否正在 `range`方式访问当前的buckets
    // 5. 是否有 `range`方式访问旧的bucket
    flags     uint8 
    
    B         uint8  // buckets 的对数 log_2
    noverflow uint16 // overflow 的 bucket 近似数
    hash0     uint32 // hash种子 计算 key 的哈希的时候会传入哈希函数
    buckets   unsafe.Pointer // 指向 buckets 数组，大小为 2^B 如果元素个数为0，就为 nil
    
    // 扩容的时候，buckets 长度会是 oldbuckets 的两倍
    oldbuckets unsafe.Pointer // bucket slice指针，仅当在扩容的时候不为nil
    
    nevacuate  uintptr // 扩容时已经移到新的map中的bucket数量
    extra *mapextra // optional fields
}
```
注意：**B 是buckets 数组的长度的对数，也就是说 buckets 数组的长度就是 2^B。bucket 里面存储了 key 和 value。**

**buckets 是一个指针，最终它指向的是一个结构体：（buckets是bmap类型的数组，数组长度是2^B）**
```
// A bucket for a Go map.
type bmap struct {
    tophash [bucketCnt]uint8
}
```
**bmap就是我们所说的桶bucket，实际上就是每个bucket固定包含8个key和value(可以查看源码bucketCnt=8).实现上面是一个固定的大小连续内存块，分成四部分：**

1. 每个条目的状态
2. 8个key值
3. 8个value值
4. 指向下个bucket的指针

桶里面会最多装 8 个key，这些key之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置）。

查看下图： B=5 表示hmap的有2^5=32个bmap：buckets是一个bmap数组，其长度为32。 每个bmap有8个key

![1](AEA8E8EC037D4100A42F5AE308A62C3A)
![1](https://github.com/uglychen/chenxun_go/blob/master/go%E6%96%87%E6%A1%A3%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/images/1.png)
## hmap的数据结构


```
   ----+-----------------+-.            bmp
   ^   |     bucket0     | |------>  +------------+
   |   +-----------------+-'         | tophash0-7 |
2^h.B  |     .......     |           +------------+
   |   +-----------------+           |   key0-7   |
   v   | bucket2^h.B - 1 |           +------------+
   ----+-----------------+           |  value0-7  |
                                     +------------+ -.
                                     |overflow_ptr|  |-----> new bucket address
```
选择这样的布局的好处：由于对齐的原因，key0/value0/key1/value1… 这样的形式可能需要更多的补齐空间，比如 map[int64]int8 ，1字节的value后面需要补齐7个字节才能保证下一个key是 int64 对齐的。

**每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 overflow 指针连接起来**

# 创建 map

```
ageMp := make(map[string]int)
// 指定 map 长度
ageMp := make(map[string]int, 8)

// ageMp 为 nil，不能向其添加元素，会直接panic
var ageMp map[string]int
```
**通过汇编语言可以看到，实际上底层调用的是 makemap 函数，主要做的工作就是初始化 hmap 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 等等**


```
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
    // 省略各种条件检查...

    // 找到一个 B，使得 map 的装载因子在正常范围内
    B := uint8(0)
    for ; overLoadFactor(hint, B); B++ {
    }

    // 初始化 hash table
    // 如果 B 等于 0，那么 buckets 就会在赋值的时候再分配
    // 如果长度比较大，分配内存会花费长一点
    buckets := bucket
    var extra *mapextra
    if B != 0 {
        var nextOverflow *bmap
        buckets, nextOverflow = makeBucketArray(t, B)
        if nextOverflow != nil {
            extra = new(mapextra)
            extra.nextOverflow = nextOverflow
        }
    }

    // 初始化 hamp
    if h == nil {
        h = (*hmap)(newobject(t.hmap))
    }
    h.count = 0
    h.B = B
    h.extra = extra
    h.flags = 0
    h.hash0 = fastrand()
    h.buckets = buckets
    h.oldbuckets = nil
    h.nevacuate = 0
    h.noverflow = 0

    return h
}
```

# 哈希函数

**map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 alginit() 中完成，位于路径：src/runtime/alg.go 下**

- [ ] hash 函数，有加密型和非加密型。
加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种；
非加密型的一般就是查找。在 map 的应用场景中，用的是查找。
选择 hash 函数主要考察的是两点：性能、碰撞概率


```
type typeAlg struct {
	// function for hashing objects of this type
	// (ptr to object, seed) -> hash
	hash func(unsafe.Pointer, uintptr) uintptr
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
}
```
typeAlg 包含两个函数，hash 函数计算类型的哈希值，而 equal 函数则计算两个类型是否“哈希相等”。

对于 string 类型，它的 hash、equal 函数如下：

```
func strhash(a unsafe.Pointer, h uintptr) uintptr {
    x := (*stringStruct)(a)
    return memhash(x.str, h, uintptr(x.len))
}

func strequal(p, q unsafe.Pointer) bool {
    return *(*string)(p) == *(*string)(q)
}

var algarray = [alg_max]typeAlg{
	alg_NOEQ:     {nil, nil},
	alg_MEM0:     {memhash0, memequal0},
	alg_MEM8:     {memhash8, memequal8},
	alg_MEM16:    {memhash16, memequal16},
	alg_MEM32:    {memhash32, memequal32},
	alg_MEM64:    {memhash64, memequal64},
	alg_MEM128:   {memhash128, memequal128},
	alg_STRING:   {strhash, strequal},
	alg_INTER:    {interhash, interequal},
	alg_NILINTER: {nilinterhash, nilinterequal},
	alg_FLOAT32:  {f32hash, f32equal},
	alg_FLOAT64:  {f64hash, f64equal},
	alg_CPLX64:   {c64hash, c64equal},
	alg_CPLX128:  {c128hash, c128equal},
}
```

# key 定位过程
**key 经过哈希计算后得到哈希值，共 64 个 bit 位（64位机，32位机就不讨论了，现在主流都是64位机），计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。还记得前面提到过的 B 吗？如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。**

例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

```
10010111 | 000011110110110010001111001010100010010110010101010 │ 01010

```

用最后的 5 个 bit 位，也就是 01010，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。

再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。最开始桶内还没有 key，新加入的 key 会找到第一个空位，放入。

buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。

![2](0B20ECE99D4245F18FD2EEEDB2A3F063)
![image](https://github.com/uglychen/chenxun_go/blob/master/go%E6%96%87%E6%A1%A3%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/images/2.png)

上图中，假定 B = 5，所以 bucket 总数就是 2^5 = 32。首先计算出待查找 key 的哈希，使用低 5 位 00110，找到对应的 6 号 bucket，使用高 8 位 10010111，对应十进制 151，在 6 号 bucket 中寻找 tophash 值（HOB hash）为 151 的 key，找到了 2 号槽位，这样整个查找过程就结束了。

如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。

我们来看下源码吧，哈哈！通过汇编语言可以看到，查找某个 key 的底层函数是 mapacess 系列函数，函数的作用类似，区别在下一节会讲到。这里我们直接看 mapacess1 函数：
```
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // ……
    
    // 如果 h 什么都没有，返回零值
    if h == nil || h.count == 0 {
        return unsafe.Pointer(&zeroVal[0])
    }
    
    // 写和读冲突
    if h.flags&hashWriting != 0 {
        throw("concurrent map read and map write")
    }
    
    // 不同类型 key 使用的 hash 算法在编译期确定
    alg := t.key.alg
    
    // 计算哈希值，并且加入 hash0 引入随机性
    hash := alg.hash(key, uintptr(h.hash0))
    
    // 比如 B=5，那 m 就是31，二进制是全 1
    // 求 bucket num 时，将 hash 与 m 相与，
    // 达到 bucket num 由 hash 的低 8 位决定的效果
    m := uintptr(1)<<h.B - 1
    
    // b 就是 bucket 的地址
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
    
    // oldbuckets 不为 nil，说明发生了扩容
    if c := h.oldbuckets; c != nil {
        // 如果不是同 size 扩容（看后面扩容的内容）
        // 对应条件 1 的解决方案
        if !h.sameSizeGrow() {
            // 新 bucket 数量是老的 2 倍
            m >>= 1
        }
        
        // 求出 key 在老的 map 中的 bucket 位置
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        
        // 如果 oldb 没有搬迁到新的 bucket
        // 那就在老的 bucket 中寻找
        if !evacuated(oldb) {
            b = oldb
        }
    }
    
    // 计算出高 8 位的 hash
    // 相当于右移 56 位，只取高8位
    top := uint8(hash >> (sys.PtrSize*8 - 8))
    
    // 增加一个 minTopHash
    if top < minTopHash {
        top += minTopHash
    }
    for {
        // 遍历 8 个 bucket
        for i := uintptr(0); i < bucketCnt; i++ {
            // tophash 不匹配，继续
            if b.tophash[i] != top {
                continue
            }
            // tophash 匹配，定位到 key 的位置
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
            // key 是指针
            if t.indirectkey {
                // 解引用
                k = *((*unsafe.Pointer)(k))
            }
            // 如果 key 相等
            if alg.equal(key, k) {
                // 定位到 value 的位置
                v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                // value 解引用
                if t.indirectvalue {
                    v = *((*unsafe.Pointer)(v))
                }
                return v
            }
        }
        
        // bucket 找完（还没找到），继续到 overflow bucket 里找
        b = b.overflow(t)
        // overflow bucket 也找完了，说明没有目标 key
        // 返回零值
        if b == nil {
            return unsafe.Pointer(&zeroVal[0])
        }
    }
}
```
函数返回 h[key] 的指针，如果 h 中没有此 key，那就会返回一个 key 相应类型的零值，不会返回 nil。

代码整体比较直接，没什么难懂的地方。跟着上面的注释一步步理解就好了。

这里，说一下定位 key 和 value 的方法以及整个循环的写法。

**b 是 bmap 的地址，这里 bmap 还是源码里定义的结构体，只包含一个 tophash 数组，经编译器扩充之后的结构体才包含 key，value，overflow 这些字段。dataOffset 是 key 相对于 bmap 起始地址的偏移：**

```
dataOffset = unsafe.Offsetof(struct {
        b bmap
        v int64
    }{}.v)
```
因此 bucket 里 key 的起始地址就是 unsafe.Pointer(b)+dataOffset。第 i 个 key 的地址就要在此基础上跨过 i 个 key 的大小；而我们又知道，value 的地址是在所有 key 之后，因此第 i 个 value 的地址还需要加上所有 key 的偏移。理解了这些，上面 key 和 value 的定位公式就很好理解了。

再说整个大循环的写法，最外层是一个无限循环，通过

```
b = b.overflow(t)
```
# map 的两种 get 操作


```
package main

import "fmt"

func main() {
    ageMap := make(map[string]int)
    ageMap["qcrao"] = 18

    // 不带 comma 用法
    age1 := ageMap["stefno"]
    fmt.Println(age1)

    // 带 comma 用法
    age2, ok := ageMap["stefno"]
    fmt.Println(age2, ok)
}
```

```
// src/runtime/hashmap.go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)

uint32	mapaccess1_fast32(t maptype, h hmap, key uint32) unsafe.Pointer
uint32	mapaccess2_fast32(t maptype, h hmap, key uint32) (unsafe.Pointer, bool)
uint64	mapaccess1_fast64(t maptype, h hmap, key uint64) unsafe.Pointer
uint64	mapaccess2_fast64(t maptype, h hmap, key uint64) (unsafe.Pointer, bool)
string	mapaccess1_faststr(t maptype, h hmap, ky string) unsafe.Pointer
string	mapaccess2_faststr(t maptype, h hmap, ky string) (unsafe.Pointer, bool)
```
# map扩容

使用哈希表的目的就是要快速查找到目标 key，然而，随着向 map 中添加的 key 越来越多，key 发生碰撞的概率也越来越大。bucket 中的 8 个 cell 会被逐渐塞满，查找、插入、删除 key 的效率也会越来越低。最理想的情况是一个 bucket 只装一个 key，这样，就能达到 O(1) 的效率，但这样空间消耗太大，用空间换时间的代价太高。

Go 语言采用一个 bucket 里装载 8 个 key，定位到某个 bucket 后，还需要再定位到具体的 key，这实际上又用了时间换空间。

当然，这样做，要有一个度，不然所有的 key 都落在了同一个 bucket 里，直接退化成了链表，各种操作的效率直接降为 O(n)，是不行的。

因此，需要有一个指标来衡量前面描述的情况，这就是装载因子。Go 源码里这样定义 装载因子

```
loadFactor := count / (2^B)
```
count 就是 map 的元素个数，2^B 表示 bucket 数量。

再来说触发 map 扩容的时机：在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：

装载因子超过阈值，源码里定义的阈值是 6.5。
overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。
通过汇编语言可以找到赋值操作对应源码中的函数是 mapassign，对应扩容条件的源码如下：  
**通过汇编语言可以找到赋值操作对应源码中的函数是 mapassign，对应扩容条件的源码如下：**

```
// src/runtime/hashmap.go/mapassign

// 触发扩容时机
if !h.growing() && (overLoadFactor(int64(h.count), h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
    }

// 装载因子超过 6.5
func overLoadFactor(count int64, B uint8) bool {
    return count >= bucketCnt && float32(count) >= loadFactor*float32((uint64(1)<<B))
}

// overflow buckets 太多
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
    if B < 16 {
        return noverflow >= uint16(1)<<B
    }
    return noverflow >= 1<<15
}
```
解释一下：

第 1 点：我们知道，每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。

第 2 点：是对第 1 点的补充。就是说在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。

不难想像造成这种情况的原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触犯第 1 点的规定，你能拿我怎么办？overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，因此出台第 2 点规定。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

对于命中条件 1，2 的限制，都会发生扩容。但是扩容的策略并不相同，毕竟两种条件应对的场景不同。

对于条件 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。而且，新 bucket 只是最大数量变为原来最大数量（2^B）的 2 倍（2^B * 2）。

对于条件 2，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。

对于条件 2 的解决方案，曹大的博客里还提出了一个极端的情况：如果插入 map 的 key 哈希都一样，就会落到同一个 bucket 里，超过 8 个就会产生 overflow bucket，结果也会造成 overflow bucket 数过多。移动元素其实解决不了问题，因为这时整个哈希表已经退化成了一个链表，操作效率变成了 O(n)。

再来看一下扩容具体是怎么做的。由于 map 扩容需要将原有的 key/value 重新搬迁到新的内存地址，如果有大量的 key/value 需要搬迁，会非常影响性能。因此 Go map 的扩容采取了一种称为“渐进式”地方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。

上面说的 hashGrow() 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 growWork() 函数中，而调用 growWork() 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil。

我们先看 hashGrow() 函数所做的工作，再来看具体的搬迁 buckets 是如何进行的。


```
func hashGrow(t *maptype, h *hmap) {
    // B+1 相当于是原来 2 倍的空间
    bigger := uint8(1)

    // 对应条件 2
    if !overLoadFactor(int64(h.count), h.B) {
        // 进行等量的内存扩容，所以 B 不变
        bigger = 0
        h.flags |= sameSizeGrow
    }
    // 将老 buckets 挂到 buckets 上
    oldbuckets := h.buckets
    // 申请新的 buckets 空间
    newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger)

    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }
    // 提交 grow 的动作
    h.B += bigger
    h.flags = flags
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
    // 搬迁进度为 0
    h.nevacuate = 0
    // overflow buckets 数为 0
    h.noverflow = 0

    // ……
}
```
[https://www.cnblogs.com/qcrao-2018/archive/2019/05/22/10903807.html](https://www.cnblogs.com/qcrao-2018/archive/2019/05/22/10903807.html)











