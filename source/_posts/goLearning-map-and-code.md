---
title: Golang map使用与源码阅读
date: 2019-03-16 10:54:41
catagories: 
- go学习笔记
tags:
- map
- go
---

<!-- toc -->

# map的使用

* 花重点
    * map是引用类型
    * map的遍历顺序是随机的
    * map元素不可寻址


```go
//声明,此时 m值为nil，字节赋值会panic
var m map[string]string
//初始化
m = make(map[string]string)
//带大小初始化，使用较少
m = make(map[string]string,5)
//初始化赋值
m1 := map[int]int{
    1:1,
    2:2,
}
//添加元素
m["a"] = "a"
//获取元素 一个返回值
//当key不存在时，返回value对应类型的零值
b := m["a"]

//获取元素 两个返回值,当ok为true时，表示key存在
if b1,ok := m["a"];ok{
    fmt.Println(b1)
}
//遍历map，与插入顺序无关
for k, v := range m{
    fmt.Println(k,v)
}

//修改元素
m["a"] = "c"
//删除元素
delete(m,"a")

//查看map大小
l := len(m)


```

# map源码阅读

##代码
* 代码基于Go1.12
* runtime/map.go
* runtime/map_fast32.go
* runtime/map_fast64.go
* runtime/map_faststr.go

## 常量定义

```go
const (
	// Maximum number of key/value pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits

	// Maximum average load of a bucket that triggers growth is 6.5.
	// Represent as loadFactorNum/loadFactDen, to allow integer math.
	loadFactorNum = 13
	loadFactorDen = 2

	// Maximum key or value size to keep inline (instead of mallocing per element).
	// Must fit in a uint8.
	// Fast versions cannot handle big values - the cutoff size for
	// fast versions in cmd/compile/internal/gc/walk.go must be at most this value.
	maxKeySize   = 128
	maxValueSize = 128

	// data offset should be the size of the bmap struct, but needs to be
	// aligned correctly. For amd64p32 this means 64-bit alignment
	// even though pointers are 32 bit.
	dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)

	// Possible tophash values. We reserve a few possibilities for special marks.
	// Each bucket (including its overflow buckets, if any) will have either all or none of its
	// entries in the evacuated* states (except during the evacuate() method, which only happens
	// during map writes and thus no one else can observe the map during that time).
	emptyRest      = 0 // this cell is empty, and there are no more non-empty cells at higher indexes or overflows.
	emptyOne       = 1 // this cell is empty
	evacuatedX     = 2 // key/value is valid.  Entry has been evacuated to first half of larger table.
	evacuatedY     = 3 // same as above, but evacuated to second half of larger table.
	evacuatedEmpty = 4 // cell is empty, bucket is evacuated.
	minTopHash     = 5 // minimum tophash for a normal filled cell.

	// map header 的标记位 flags
	iterator     = 1 // 在 buckets 使用迭代器
	oldIterator  = 2 // 在 oldbuckets 上使用迭代器
	hashWriting  = 4 // 有协程在写map
	sameSizeGrow = 8 // the current map growth is to a new map of the same size

	// sentinel bucket ID for iterator checks
	noCheck = 1<<(8*sys.PtrSize) - 1
)
```

## map的结构

```go

// A header for a Go map.
type hmap struct {
	count     int // map的大小，供len读取
	flags     uint8 //标记位 占位使用 使用了4位 含义见常量定义标记位部分
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // 溢出的bucket个数
	hash0     uint32 // hash种子，map初始化时随机生成，保证hash随机性

	buckets    unsafe.Pointer // 大小为2^B的buckets数组的地址，当map大小为0时为nil 
	oldbuckets unsafe.Pointer // oldbuckets 旧桶地址，大小为当前的1/2，在扩容时不为nil
	nevacuate  uintptr        // 当前迁移的数目，当bucket地址小于该值时，表示该bucket以迁移过

	extra *mapextra // optional fields  
}

// mapextra用来保存一些不会在所有的map中都应用的特殊变量
type mapextra struct {
	// If both key and value do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
    //当且仅当没有使用指针作为map的key、value时，overflow、oldoverflow分别为hmap.buckets、hamp.oldbuckets
    //保存溢出桶
    // The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

// 桶的实现
type bmap struct {
    // tophash 只保存桶内key的hash值的高八位
    //若tophash[0]<minTopHash,则tophash[0]表示该桶当前的迁移状态
	tophash [bucketCnt]uint8
	// 后面是bucketCnt（8）个key和8个value，将key和value聚集保存（key/key/.../value/value/...），而不是key/value/key/value...形式保存，可以避免例如保存map[int64]int8这种结构时所需要进行的padding带来的共建浪费
	// Followed by an overflow pointer.
}

```

## map的创建

```go
//hint > int值范围时使用，将hint改为0
func makemap64(t *maptype, hint int64, h *hmap) *hmap {
	if int64(int(hint)) != hint {
		hint = 0
	}
	return makemap(t, int(hint), h)
}

//当使用make(map[type]type)、编译时发现map需使用最大bucketCnt、map需要分配在堆上时使用
func makemap_small() *hmap {
	h := new(hmap)
	h.hash0 = fastrand()//获取随机的hash种子
	return h
}

// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.kind&kindNoPointers == 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```



# 参考
* https://www.jianshu.com/p/aa0d4808cbb8