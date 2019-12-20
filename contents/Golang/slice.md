### slice 使用技巧总结

- slice 的截取
```go
a := make([]int, 10)
b := a[2:7] // 不包含第七个元素
fmt.Println(cap(b)) // 8
fmt.Println(len(b)) // 5
```
- slice 排序
```go
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

```go
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
```go
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
```go
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
```go
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
```go
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
```go
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