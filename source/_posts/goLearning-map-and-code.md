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
    * key值一定是可比较的(可理解为支持==)
    * map不是并发安全的


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

## 代码来源

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

	// tophash 的可能值
	// 当tophash对应的槽位没有元素时，用来标志当前bucket的一些状态
	// 每一个bucket(包括其链接的overflow bucket)，其元素或者都已迁移完毕，或者未迁移
	// 不存在部分迁移，部分未迁移(map写操作执行evacuate() 方法的过程中除外)
	emptyRest      = 0 // 该槽位为空，且该bucket其后的其他槽位以及该bucket后链接的overflow bucket没有非空槽位
	emptyOne       = 1 // 该槽位为空
	evacuatedX     = 2 // key/value 合法，该元素已迁移到较大hash table的前半部分
	evacuatedY     = 3 // key/value 合法，该元素已迁移到较大hash table的后半部分
	evacuatedEmpty = 4 // 槽位为空，该bucket已迁移完
	minTopHash     = 5 // tophash 可用于保存的最小hash值

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
	// 后面是bucketCnt（8）个key和8个value
	//将key和value聚集保存（key/key/.../value/value/...）
	//而不是key/value/key/value...形式保存，
	//可以避免例如保存map[int64]int8这种结构时所需要进行的padding带来的共建浪费
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


//make(map[k]v,hint)的实现
//若编译器检测到map或者第一个桶可以分配在栈上，则h非nil(h.buckets也可能非nil)
//map可以直接在h上创建
//若h.buchets非nil，则可直接用做第一个桶
func makemap(t *maptype, hint int, h *hmap) *hmap {
	//满足hint所需的内存过大，改为动态分配
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// 初始化 Hmap
	if h == nil {
		h = new(hmap)
	}
	//初始化hash的随机种子
	h.hash0 = fastrand()

	//计算保存申请元素个数所需的B大小
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// 为hash table分配内存
	// 若B == 0 则延迟分配 (调用mapassign时分配)
	// 对于较大的hint，分配内存会有一定的时间消耗
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		// 初始化map中的mapextra，管理未使用溢出桶的位置
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}


// makeBucketArray 初始化map中buckets所需的数组，最小分配 1 << b 个
// dirtyalloc应该为nil或者之前使用相同的t、b参数调用makeBucketArray创建的数组
// 若dirtyalloc为nil，则分配新数组，否则将dirtyalloc清空复用
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	// 计算所需的bucket数目 1 << b
	base := bucketShift(b)
	nbuckets := base
	// 对于较小的b，计算所需溢出桶作用不大，避免计算
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		// 计算需要增加的overflow bucket数目
		nbuckets += bucketShift(b - 4)
		// 地址对齐
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	//创建新数组或者清空复用dirtyalloc
	if dirtyalloc == nil {
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc 可能非空，所以要做一遍清除数据操作
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.kind&kindNoPointers == 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

	//存在溢出桶时的特殊处理
	if base != nbuckets {
		// 计算出未使用溢出桶的地址，以便之后管理使用
		// 使用一个非空的指针标记最后一个溢出桶，这里使用的时buckets
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```

## map 增加元素

```go
// 与mapaccess逻辑基本一致，但是当key不存在时，会为其分配一个槽位
// m[k] = v 时使用
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	//不可给为nil的map添加元素
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	//竞态、读写并发检查
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.key.size)
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// 修改写入标志位，因为其操作没有加锁，所以map并不是并发安全的
	// 在alg.hash后才置写入标志位，是因为alg.hash()有可能会panic，则此时其实没有发生写入
	h.flags ^= hashWriting

	// 对于传入hint为0的，h.buckets延迟初始化
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
	//确定在哪一个bucket，取hash值得h.B - 1 位 作为bucket得位置 
	bucket := hash & bucketMask(h.B)
	// 判断h.oldbuckets是否为nil，不为nil，则表示map在扩容中
	// 若map在扩容中，则将相关的oldbucket迁移到当前的bucket
	if h.growing() {
		// 执行oldbucket的迁移
		growWork(t, h, bucket)
	}
	// 计算key所在bucket地址
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
	// 取hash的高八位
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var val unsafe.Pointer
bucketloop:
	for {
		//遍历该bucket的bucketCnt个元素
		for i := uintptr(0); i < bucketCnt; i++ {
			// 先用topHash 简单比较
			if b.tophash[i] != top {
				// 发现空位置，记录tophash和key位置备用
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				// 没有其他空位置，跳出循环，使用该备选位置
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			// tophash 一致，找到该k值，与输入key比较
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 存储的k与输入key不同，继续遍历查找
			if !alg.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			// 获取key对应的value地址，跳转到done
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done
		}
		// 获取该bucket的溢出bucket，继续遍历
		ovf := b.overflow(t)
		// 无溢出bucket，跳出
		if ovf == nil {
			break
		}
		b = ovf
	}

	// 未找到key，为key创建键值对

	// 当此时负载因子到达阈值或者溢出bucket过多，则启动map的扩容
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // map扩容，hash发生变化，重新计算
	}

	// 未发现可用槽位
	if inserti == nil {
		// 当前的bucket都是满的，未发现空槽位，则获取一个未使用的bucket
		// 若还有未使用的overflow bucket，
		// 否则创建一个新bucket，并对h.noverflow+1
		// 将该bucket链在当前遍历的bucket后，并返回该bucket
		newb := h.newoverflow(t, b)
		// 获取插入槽位的tophash、key、value地址
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// 若key、value是指针类型，创建新对象并保存
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectvalue() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(val) = vmem
	}
	typedmemmove(t.key, insertk, key)
	// 保存tophash
	*inserti = top
	// map的元素数+1
	h.count++

done:
	//发生了并发写，因为对写入标志位的修改是采用异或方式，发生两个协程同时写入时，标志位会变0
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectvalue() {
		val = *((*unsafe.Pointer)(val))
	}
	return val
}
```

## map 元素访问

```go
//a := m["a"] 单返回值 方法使用，key不存在则返回零值
//与mapaccess2实现基本一致，除了两处指针地址计算外
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	//是否打开竞态检查
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		//https://github.com/golang/go/issues/23734
		//当使用不可比较值作为key访问map，即使map为空，也panic
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		//返回零值
		return unsafe.Pointer(&zeroVal[0])
	}
	//发生并发读写
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	//计算key的hash值
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	//此处mapaccess2实现是直接计算
	// 计算该hash值在h.buckets中所对应的bucket
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	// 遍历oldbucket，若发现该hash所对应的bucket未迁移，改为在h.oldbuckets中查找
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	//取hash值得高八位
	top := tophash(hash)
bucketloop:
	// 遍历该bucket及其关联的overflow bucket
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				// 空bucket，跳出
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			// 判断保存的k与输入的是否一致，一致则返回value
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				if t.indirectvalue() {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
	}
	// 未找到key，返回对应value的零值
	return unsafe.Pointer(&zeroVal[0])
}

//v,ok := m[k] 两返回值访问调用，key不存在时ok为false，v为对应类型零值
//除了返回值多一个，其他与mapaccess1基本一致
//那么为什么mapaccess1不封装mapaccess2，而是要再实现一遍?
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess2)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0]), false
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	//此处与mapaccess1实现不一致，mapaccess1使用的为add(),该方法其实就是实现的指针地址加的逻辑
	/*
	//runtime/stubs.go
	// Should be a built-in for unsafe.Pointer?
	//go:nosplit
	func add(p unsafe.Pointer, x uintptr) unsafe.Pointer {
		return unsafe.Pointer(uintptr(p) + x)
	}
	*/
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				if t.indirectvalue() {
					v = *((*unsafe.Pointer)(v))
				}
				return v, true
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0]), false
}

//返回值k，v,供map的迭代器使用
//基本逻辑与mapaccess1、mapaccess2一致
func mapaccessK(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, unsafe.Pointer) {
	if h == nil || h.count == 0 {
		return nil, nil
	}
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(unsafe.Pointer(uintptr(c) + (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				if t.indirectvalue() {
					v = *((*unsafe.Pointer)(v))
				}
				return k, v
			}
		}
	}
	return nil, nil
}

// 返回零值指针 零值类型占用内存过大时使用  /cmd/compile/internal/gc/walk.go:1107
func mapaccess1_fat(t *maptype, h *hmap, key, zero unsafe.Pointer) unsafe.Pointer {
	v := mapaccess1(t, h, key)
	if v == unsafe.Pointer(&zeroVal[0]) {
		return zero
	}
	return v
}
// 同上 /cmd/compile/internal/gc/walk.go:770
func mapaccess2_fat(t *maptype, h *hmap, key, zero unsafe.Pointer) (unsafe.Pointer, bool) {
	v := mapaccess1(t, h, key)
	if v == unsafe.Pointer(&zeroVal[0]) {
		return zero, false
	}
	return v, true
}

```

## map 元素删除

```go
//查找过程同 增加元素 mapassign
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapdelete)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write (delete).
	h.flags ^= hashWriting

	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !alg.equal(key, k2) {
				continue
			}
			// 若key、value为指针，仅将其置为nil，不释放其指向内容,示例
			/*
			package main

			import(
					"fmt"
			)
			type a struct{
					A int
			}
			func main(){
					m := make(map[*a]*a)
					b := &a{1}
					c := &a{2}
					m[c]=b
					fmt.Println(m[c])
					delete(m,c)
					fmt.Println(m[c])
					fmt.Println(c.A)
					fmt.Println(b.A)
			}
			*/
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.kind&kindNoPointers == 0 {
				memclrHasPointers(k, t.key.size)
			}
			v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			if t.indirectvalue() {
				*(*unsafe.Pointer)(v) = nil
			} else if t.elem.kind&kindNoPointers == 0 {
				memclrHasPointers(v, t.elem.size)
			} else {
				memclrNoHeapPointers(v, t.elem.size)
			}
			// 将tophash置为该槽位为空
			b.tophash[i] = emptyOne
			// If the bucket now ends in a bunch of emptyOne states,
			// change those to emptyRest states.
			// It would be nice to make this a separate function, but
			// for loops are not currently inlineable.
			// 若之后无bucket或者元素，将前一个元素的标志位置为emptyRest
			if i == bucketCnt-1 {
				if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
					goto notLast
				}
			} else {
				if b.tophash[i+1] != emptyRest {
					goto notLast
				}
			}
			for {
				b.tophash[i] = emptyRest
				if i == 0 {
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}
	//检查并发写
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}

```

## map 的迭代器

```go
// A hash iteration structure.
// If you modify hiter, also change cmd/compile/internal/gc/reflect.go to indicate
// the layout of this structure.
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/internal/gc/range.go).
	value       unsafe.Pointer // Must be in second position (see cmd/internal/gc/range.go).
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // bucket iteration started at
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}

// mapiterinit initializes the hiter struct used for ranging over maps.
// The hiter struct pointed to by 'it' is allocated on the stack
// by the compilers order pass or on the heap by reflect_mapiterinit.
// Both need to have zeroed hiter since the struct contains pointers.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiterinit))
	}

	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
	}
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.kind&kindNoPointers != 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}

func mapiternext(it *hiter) {
	h := it.h
	if raceenabled {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiternext))
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket
	alg := t.key.alg

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.value = nil
			return
		}
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
		if checkBucket != noCheck && !h.sameSizeGrow() {
			// Special case: iterator was started during a grow to a larger size
			// and the grow is not done yet. We're working on a bucket whose
			// oldbucket has not been evacuated yet. Or at least, it wasn't
			// evacuated when we started the bucket. So we're iterating
			// through the oldbucket, skipping any keys that will go
			// to the other new bucket (each oldbucket expands to two
			// buckets during a grow).
			if t.reflexivekey() || alg.equal(k, k) {
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := alg.hash(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// Hash isn't repeatable if k != k (NaNs).  We need a
				// repeatable and randomish choice of which direction
				// to send NaNs during evacuation. We'll use the low
				// bit of tophash to decide which way NaNs go.
				// NOTE: this case is why we need two evacuate tophash
				// values, evacuatedX and evacuatedY, that differ in
				// their low bit.
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || alg.equal(k, k)) {
			// This is the golden data, we can return it.
			// OR
			// key!=key, so the entry can't be deleted or updated, so we can just return it.
			// That's lucky for us because when key!=key we can't look it up successfully.
			it.key = k
			if t.indirectvalue() {
				v = *((*unsafe.Pointer)(v))
			}
			it.value = v
		} else {
			// The hash table has grown since the iterator was started.
			// The golden data for this key is now somewhere else.
			// Check the current hash table for the data.
			// This code handles the case where the key
			// has been deleted, updated, or deleted and reinserted.
			// NOTE: we need to regrab the key as it has potentially been
			// updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
			rk, rv := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.value = rv
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

## map的清除

* 编译器发现代码尝试使用range将map中所有key删除时，会优化为调用mapclear()

```go
// code /cmd/compile/internal/gc/walk.go:151
// walkrange transforms various forms of ORANGE into
// simpler forms.  The result must be assigned back to n.
// Node n may also be modified in place, and may also be
// the returned node.
func walkrange(n *Node) *Node {
	if isMapClear(n) {
		m := n.Right
		lno := setlineno(m)
		n = mapClear(m)
		lineno = lno
		return n
	}
	// code
}

// code /cmd/compile/internal/gc/walk.go:458
// isMapClear checks if n is of the form:
//
// for k := range m {
//   delete(m, k)
// }
//
// where == for keys of map m is reflexive.
func isMapClear(n *Node) bool {
	// code
	return true
}
```

```go
// mapclear 清除map中所有的key
func mapclear(t *maptype, h *hmap) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapclear)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
	}

	if h == nil || h.count == 0 {
		return
	}

	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

	h.flags ^= hashWriting

	h.flags &^= sameSizeGrow
	h.oldbuckets = nil
	h.nevacuate = 0
	h.noverflow = 0
	h.count = 0

	// Keep the mapextra allocation but clear any extra information.
	if h.extra != nil {
		*h.extra = mapextra{}
	}

	// makeBucketArray clears the memory pointed to by h.buckets
	// and recovers any overflow buckets by generating them
	// as if h.buckets was newly alloced.
	_, nextOverflow := makeBucketArray(t, h.B, h.buckets)
	if nextOverflow != nil {
		// If overflow buckets are created then h.extra
		// will have been allocated during initial bucket creation.
		h.extra.nextOverflow = nextOverflow
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
```

## map的扩容
* 当负载因子到达阈值或者overflow bucket过多时，触发map的扩容
* 负载因子到达阈值，两倍扩容
* overflow bucket过多，等大小“扩容”(sameSizeGrow),类似内存管理的复制算法，将槽位连续利用
  
### 扩容流程
* 仅当map增加元素时才会触发扩容的启动（负载因子到达阈值或者overflow bucket过多），会将原有的h.buckets转移到h.oldbuckets并初始化新的h.buckets，初始化迁移进度变量h.nevacuate
* 在mapassign和mapdelete时会迁移元素，将本次操作key所涉及的bucket及其overflow bucket都迁移，再根据h.nevacuate位置，迁移一个bucket及其overflow bucket，并更新h.nevacuate

```go
//该方法仅新建bucket，并将原有buckets转移到oldbuckets，真正的元素迁移由growWork() 和 evacuate()执行
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
	// 判断采用何种扩容方式
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}

	oldbuckets := h.buckets
	// 创建新的buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	// 将bucket的迭代标志位转移到oldbucket迭代标志位
	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	// 转移当前的orverflow bucket到oldoverflow bucket，并置为nil
	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	// 处理新的未使用的overflow bucket
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

}

// 判断负载因子是否到达阈值
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// tooManyOverflowBuckets reports whether noverflow buckets is too many for a map with 1<<B buckets.
// Note that most of these overflow buckets must be in sparse use;
// if use was dense, then we'd have already triggered regular map growth.
// 判断overflow bucket是否过多，过多说明大量元素集中在特定的一个或者几个bucket链表上
// 导致这一个或者几个bucket链表不断溢出，若未触发扩容，则愿意可能是在bucket链上存在大量空槽位
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}

// growing reports whether h is growing. The growth may be to the same size or bigger.
// 若h.oldbuckets不为nil，说明map在调整
func (h *hmap) growing() bool {
	return h.oldbuckets != nil
}

// sameSizeGrow reports whether the current growth is to a map of the same size.
func (h *hmap) sameSizeGrow() bool {
	return h.flags&sameSizeGrow != 0
}

// noldbuckets calculates the number of buckets prior to the current map growth.
// 计算h.oldbuckets中需要迁移的bucket数目
// 若为sameSizeGrow，则和h.buckets相同，否则为1/2
func (h *hmap) noldbuckets() uintptr {
	oldB := h.B
	if !h.sameSizeGrow() {
		oldB--
	}
	return bucketShift(oldB)
}

// oldbucketmask provides a mask that can be applied to calculate n % noldbuckets().
func (h *hmap) oldbucketmask() uintptr {
	return h.noldbuckets() - 1
}

// 给出一个bucket，判断该bucket是否已迁移
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > emptyOne && h < minTopHash
}

// map增加元素时调用，迁移当前要使用的bucket，并再迁移一个其他的bucket
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}

// 给出一个bucket，判断该bucket是否已迁移
func bucketEvacuated(t *maptype, h *hmap, bucket uintptr) bool {
	b := (*bmap)(add(h.oldbuckets, bucket*uintptr(t.bucketsize)))
	return evacuated(b)
}

// evacDst is an evacuation destination.
type evacDst struct {
	b *bmap          // current destination bucket
	i int            // key/val index into b
	k unsafe.Pointer // pointer to current key storage
	v unsafe.Pointer // pointer to current value storage
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	// 若该bucket未迁移
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		// 初始化迁移的目的地地址，若为扩大迁移，则当前的bucket可能会映射到较大hashtable的前后两段的相同位置
		// 等大小迁移则不会，因此等大小迁移不需要初始化后一段的迁移目的地址
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.v = add(x.k, bucketCnt*uintptr(t.keysize))

		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.v = add(y.k, bucketCnt*uintptr(t.keysize))
		}

		// 迁移当前bucket所在的链表
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				top := b.tophash[i]
				// 该槽位为空则标记并继续遍历下一个槽位
				if isEmpty(top) {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// 判断需要将该元素转移到两倍扩容后hashtable的前后哪一部分
					hash := t.key.alg.hash(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.alg.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						// 若未被迭代，则根据hash值用作确定bucket的低位部分的最高位判断
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				// 意义是什么，防御式编程，防止常量被修改？
				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				// 若当前迁移目标bucket的槽位索引达到槽数，
				// 即当前bucket已存满，给该bucket后链接上overflow bucket
				// 并将当前元素的迁移目的槽位设置为新bucket的第一个槽位
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				// 做dst.i&(bucketCnt-1)的意义何在？ 上面已经处理过dst.i了
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				// 转移保存key/value
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy value
				}
				if t.indirectvalue() {
					*(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v)
				} else {
					typedmemmove(t.elem, dst.v, v)
				}
				// 转移目的槽位id+1
				dst.i++
				// 此处超过当前槽位的key/value地址也可以，当转移元素时会根据dst.i检查
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.v = add(dst.v, uintptr(t.valuesize))
			}
		}
		// 清理oferflow bucket
		if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	// 若当前处理的bucket为待顺序迁移的bucket，则更新h的整体迁移状态
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}

func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
	h.nevacuate++
	// Experiments suggest that 1024 is overkill by at least an order of magnitude.
	// Put it in there as a safeguard anyway, to ensure O(1) behavior.
	// 检查后续的bucket是否存在连续已迁移的，存在则更新迁移进度，避免重复处理已迁移bucket
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
	// 迁移完毕
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
		h.flags &^= sameSizeGrow
	}
}
```




# 参考
* https://www.jianshu.com/p/aa0d4808cbb8
* https://blog.csdn.net/u011957758/article/details/82846609
* https://blog.csdn.net/i6448038/article/details/82057424