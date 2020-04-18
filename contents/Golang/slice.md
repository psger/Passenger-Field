### slice 使用技巧及原理

- slice 的截取
```golang
a := make([]int, 10)
b := a[2:7] // 不包含第七个元素
fmt.Println(cap(b)) // 8
fmt.Println(len(b)) // 5
```
- slice 截取第二个例子
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
- slice 排序
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
- 完美的 clone 一个切片
```go
// 普通方式
a := []int{1,2,3,4,5,6,7,8}
b := make([]int, len(a))
copy(b, a)

// 完美方式（完美吗？）
 b := append(a[:0:0], a...)
```

第一轮基准测试

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
第二轮基准测试
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
第三轮基准测试
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
- append 与 copy
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

- append 扩容
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

看个例子：
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