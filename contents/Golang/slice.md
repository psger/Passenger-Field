## slice 使用技巧及原理

### slice 的三种创建方式
- 字面量创建
```go
s1 := []int{1,2,3,4}
```
- 使用 make 创建
```go
s2 := make([]int, 10, 20)
```
- 使用 new 关键字创建
```go
s3 := new([]int)
```
- 直接声明
```go
var s4 []int
```

### nil slice 与 empty slice
声明方式创建的 slice 为 nil slice， `var s []int`。使用 make 创建的是 empty slice。二者的区别是：

| slice struct | nil slice | empty slice |
| :-----| ----: | :----: |
| array | 0 | 0xc40234bda0 |
| len | 0 | 0 |
| cap | 0 | 0 |

### slice 的截取
- slice 截取第一个 🌰
```golang
a := make([]int, 10)
b := a[2:7] // 不包含第七个元素
fmt.Println(cap(b)) // 8
fmt.Println(len(b)) // 5
```
- slice 截取第二个 🌰
```golang
func main() {
    orderLen := 5
    order := make([]uint16, 2 * orderLen)

    pollorder := order[:orderLen:orderLen]
    lockorder := order[orderLen:][:orderLen:orderLen]

    fmt.Println("len(pollorder) = ", len(pollorder)) // 5
    fmt.Println("cap(pollorder) = ", cap(pollorder)) // 5
    fmt.Println("len(lockorder) = ", len(lockorder)) // 5
    fmt.Println("cap(lockorder) = ", cap(lockorder)) // 5
}
```
### slice 排序
```golang
func main() {
    ps := []struct {
        name string
        age  int
    }{
        {"larry", 19},
        {"jackey", 18},
        {"lucy", 20},
    }
    // keeping the original order of equal elements
    sort.SliceStable(ps, func(i, j int) bool {
        return ps[i].age < ps[j].age
    })
    fmt.Println(ps)
}
```
### 完美的 clone 一个切片
```go
// 普通方式
a := []int{1,2,3,4,5,6,7,8}
b := make([]int, len(a))
copy(b, a)

// 完美方式（完美吗？）
 b := append(a[:0:0], a...)
```

- 第一轮基准测试

```golang
func BenchmarkSlice1(b *testing.B) {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13}

	for i := 0; i < b.N; i++ {
        c := make([]int, len(a))
		copy(c, a)
	}
}

func BenchmarkSlice2(b *testing.B) {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13}
	for i := 0; i < b.N; i++ {
		_ = append(a[:0:0], a...)
	}
}
// goos: linux
// goarch: amd64
// BenchmarkSlice1-4       29188491                41.8 ns/op
// BenchmarkSlice2-4       21514822                49.6 ns/op
// PASS
// ok      _/home/t04884/Documents/tmp     2.391s
```
- 第二轮基准测试
```golang
func BenchmarkSlice1(b *testing.B) {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13}
	c := make([]int, len(a))
	for i := 0; i < b.N; i++ {
		copy(c, a)
	}
}

func BenchmarkSlice2(b *testing.B) {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13}
	for i := 0; i < b.N; i++ {
		_ = append(a[:0:0], a...)
	}
}
// goos: linux
// goarch: amd64
// BenchmarkSlice1-4       223767306                5.29 ns/op
// BenchmarkSlice2-4       24949081                48.0 ns/op
// PASS
// ok      _/home/t04884/Documents/tmp     2.991s
```
- 第三轮基准测试
```golang
func BenchmarkSlice1(b *testing.B) {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13}
	c := []int{}
	for i := 0; i < b.N; i++ {
		copy(c, a)
	}
}

func BenchmarkSlice2(b *testing.B) {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13}
	for i := 0; i < b.N; i++ {
		_ = append(a[:0:0], a...)
	}
}
// goos: linux
// goarch: amd64
// BenchmarkSlice1-4       1000000000               0.287 ns/op
// BenchmarkSlice2-4       24956604                49.6 ns/op
// PASS
// ok      _/home/t04884/Documents/tmp     1.610s
```

这说明了啥？？
- 创建一个 slice 时最好指定容量
```golang
func BenchmarkTestAppend1(b *testing.B) {
	a := []string{}
	for i := 0; i < b.N; i++ {
		a = append(a, "hello")
	}

	fmt.Println(len(a))
}

func BenchmarkTestAppend2(b *testing.B) {
	a := make([]string, b.N)
	for i := 0; i < b.N; i++ {
		a = append(a, "hello")
	}
	fmt.Println(len(a))
}
// goos: linux
// goarch: amd64
// BenchmarkTestAppend1-4          100
// 10000
// 1000000
// 10575552
// 10575552              1196 ns/op
// 2
// BenchmarkTestAppend2-4          200
// 20000
// 2000000
// 24707862
// 12353931              1876 ns/op
// PASS
// ok      _/home/t04884/Documents/tmp     36.105s
```
不知道为什么我测出来的是这样的:(
- 使用 slice 作为缓冲来拼接字符串
```golang
package slice

import (
    "strconv"
    "unsafe"
)

func SliceInt2String1(s []int) string {
    if len(s) < 1 {
        return ""
    }

    ss := strconv.Itoa(s[0])
    for i := 1; i < len(s); i++ {
        ss += "," + strconv.Itoa(s[i])
    }

    return ss
}

func SliceInt2String2(s []int) string {
    if len(s) < 1 {
        return ""
    }

    b := make([]byte, 0, 256)
    b = append(b, strconv.Itoa(s[0])...)
    for i := 1; i < len(s); i++ {
        b = append(b, ',')
        b = append(b, strconv.Itoa(s[i])...)
    }

    return string(b)
}

func SliceInt2String3(s []int) string {
    if len(s) < 1 {
        return ""
    }

    b := make([]byte, 0, 256)
    b = append(b, strconv.Itoa(s[0])...)
    for i := 1; i < len(s); i++ {
        b = append(b, ',')
        b = append(b, strconv.Itoa(s[i])...)
    }

    return *(*string)(unsafe.Pointer(&b))
}
// goos: darwin
// goarch: amd64
// pkg: github.com/thinkeridea/example/slice
// BenchmarkSliceInt2String1-8        3000000           461 ns/op         144 B/op           9 allocs/op
// BenchmarkSliceInt2String2-8       20000000           117 ns/op          32 B/op           1 allocs/op
// BenchmarkSliceInt2String3-8       10000000           144 ns/op         256 B/op           1 allocs/op
// PASS
// ok      github.com/thinkeridea/example/slice    5.928s
```
### append 与 copy
- append 函数返回值是一个新的 slice，Go 编译器不允许调用了 append 函数后不使用返回值。
- 为什么 append 操作扩容时开销很大
    - 扩容后内存地址会发生改变，并且把原有的复制到新分配的内存中，旧的内存会被回收
    - 切片 a 创建一个子 b，这时候 a 发生了 append 操作扩容了，也就是内存地址发生了改变，那么 b 的底层指向的哪一块内存呢？
```golang
func main() {
	t := make([]int, 10)
	b := t[0:7]
	fmt.Printf("addr:%p \t\tlen:%v content:%v\n", b, len(b), b)
	fmt.Printf("addr:%p \t\tlen:%v content:%v\n", t, len(t), t)
	for i := 0; i < 10; i++ {
		t = append(t, 1)
	}
	fmt.Printf("addr:%p \t\tlen:%v content:%v\n", t, len(t), t)
	fmt.Printf("addr:%p \t\tlen:%v content:%v\n", b, len(b), b)
}
// addr:0xc0000a4000               len:7 content:[0 0 0 0 0 0 0]
// addr:0xc0000a4000               len:10 content:[0 0 0 0 0 0 0 0 0 0]
// addr:0xc0000ac000               len:20 content:[0 0 0 0 0 0 0 0 0 0 1 1 1 1 1 1 1 1 1 1]
// addr:0xc0000a4000               len:7 content:[0 0 0 0 0 0 0]
```
从上面代码运行结果可以看到，扩容之后的切片指向了新的地址，之前的子切片还是指向旧的地址。所以之后对父切片的操作不会影响子切片。

### append 扩容
- 当切片底层数组大小小于 1024 时，扩容翻倍。
- 当切片底层数组大小大于 1024 时，扩容 25%。

实际是这样吗？看 slice 扩容的[源码](https://github.com/golang/go/blob/master/src/runtime/slice.go#L76)：
```go
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
    // et.size 代表 slice 中一个元素的大小
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size) // et.size 代表 slice 中一个元素的大小
	}
}
```
实际上，还执行了：
```go
capmem = roundupsize(capmem)
newcap = int(capmem / et.size)
```

`roundupsize()` [方法](https://github.com/golang/go/blob/master/src/runtime/msize.go#L13:1)是干啥的？进行内存对齐。之前已经看过内存对齐相关的内容。
```go
// Returns size of the memory block that mallocgc will allocate if you ask for the size.
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize {
		if size <= smallSizeMax-8 {
			return uintptr(class_to_size[size_to_class8[divRoundUp(size, smallSizeDiv)]])
		} else {
			return uintptr(class_to_size[size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return alignUp(size, _PageSize)
}
```

- 看个例子：
```go
s := []int{1,2}
s = append(s,4,5,6)
fmt.Printf("%d %d",len(s),cap(s))
```
简单的看扩容后应该是 5，8。2 翻倍 4，4 不够再翻倍为 8。可是实际运行结果是，5，6。
 _MaxSmallSize 的值在 64 位 macos 上是 32«10，也就是 2 的 15 次方，32k。golang 事先生成了一个内存对齐表。通过查找 (size+7) » 3，也就是需要多少个 8 字节，然后到 class_to_size 中寻找最小内存的大小。承接上文的 size，应该是 40（5 * 8），size_to_class8 的内容是这样的：

size_to_class8：1 1 2 3 3 4 4 5 5 6 6 7 7 8 8 9 9 10 10 11 11...
       查表得到的数字是 4，而 class_to_size 的内容是这样的：

class_to_size：0 8 16 32 48 64 80 96 112 128 144 160 176 192 208 224 240 256...
       因此得到最小的对齐内存是48字节。完成内存对齐计算后，重新计算应有的容量，也就是48/8 = 6。扩容得到的容量就是6了。
这里没有弄明白的是，为什么要先查内存对齐表再查 class_to_size。

### append 为什么需要保留返回值
- 先看一段简单的代码

```go
type Slice []int

func (A Slice) Append(value int) {
	A = append(A, value)
}

func main() {
	mSlice := make(Slice, 10, 20)
	mSlice.Append(5)
	fmt.Println(mSlice) // 00000000...
}
```
为什么输出的结构中没有 5 呢？根本原因是 append 后的 slice 已经不是原来的 slice 了。
打印出 append 返回的 slice 与 A 的指针是否相同：
```go
func (A Slice)Append(value int) {
	A1 := append(A, value)
	fmt.Printf("%p\n%p\n",A,A1)
}
```
可以发现返回的指针相同，那为什么说 append 返回的 slice 不是原来的 slice 呢？
看 slice 的数据结构 sliceHeader：
```go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```
可以发现，虽然指向的底层数组可以一样，但是只要 cap 或者 len 有一个不同都不算是同一个 slice。append 之后，slice 的 len 发生了改变，所以与原来的 slice 不是同一个 slice。  
就算是 执行了 `A = append(A, value)` 也只是拷贝了一个副本，原来的 A 的改变只在 append 内生效，mSlice 本身没有改变。如果把 mSlice 的 cap 改为 10，append 时会发生扩容，指向底层数组的指针也会发生改变。  
```go
func (A Slice)Append(value int) {
	A1 := append(A, value)

	sh:=(*reflect.SliceHeader)(unsafe.Pointer(&A))
	fmt.Printf("A Data:%d,Len:%d,Cap:%d\n",sh.Data,sh.Len,sh.Cap)

	sh1:=(*reflect.SliceHeader)(unsafe.Pointer(&A1))
	fmt.Printf("A1 Data:%d,Len:%d,Cap:%d\n",sh1.Data,sh1.Len,sh1.Cap)
}

func main() {
	mSlice := make(Slice, 10, 10)
	mSlice.Append(5)
	fmt.Println(mSlice)
}

// A  Data:824633835680,Len:10,Cap:10
// A1 Data:824634204160,Len:11,Cap:20
```

### 为什么 nil slice 可以直接 append
其实 nil slice 或者 empty slice 都是可以通过调用 append 函数来获得底层数组的扩容。最终都是调用 mallocgc 来向 Go 的内存管理器申请到一块内存，然后再赋给原来的 nil slice 或 empty slice，然后摇身一变，成为“真正”的 slice 了。

### 传 slice 和 slice 指针有什么区别

### ref
- [slice 扩容规则](https://jodezer.github.io/2017/05/golangSlice%E7%9A%84%E6%89%A9%E5%AE%B9%E8%A7%84%E5%88%99)
- [Go语言slice的本质-SliceHeader](https://juejin.im/post/5c2a1e446fb9a049df242997)
- [深度解密 Go 语言之 slice](https://mp.weixin.qq.com/s/MTZ0C9zYsNrb8wyIm2D8BA)