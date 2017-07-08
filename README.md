**Assumptions**
=============================

> 1. 以下代码均在test.go中测试
> 2. 一切测试从简, 保证代码量尽可能小, 尽可能容易理解
> 3. 不推荐用的或奇技淫巧只给例子

**************************************************

**<a name="builtin-identifiers">Builtin Identifiers</a>**
=============================

**See `$GOROOT/src/builtin/builtin.go`**

### **<a name="byte-is-uint8">a. byte是uint8的别名</a>**

```go
package main

func main() {
    var u uint8 = '1'
    var b byte = u
    println(b) // prints: 49
}
```

### **<a name="rune-is-int32">b. rune是int32的别名</a>**

```go
package main

func main() {
    var is = []int32{'你', '好'}
    var rs []rune = is
    println(rs[0], rs[1]) // prints: 20320 22909
}
```

### **<a name="int-bits-sys">c. int是多少位看系统</a>**

> 有资料称:

```
自 Go1.1 后，int，uint 的尺寸统一是 64bits，即使是在 32bits 平台下。
```

```go
package main

import (
	"unsafe"
)

func main() {
	var (
		a int32
		b int64
		c int
	)

	println(unsafe.Sizeof(a)) // prints: 4
	println(unsafe.Sizeof(b)) // prints: 8
	println(unsafe.Sizeof(c)) // prints: ?(见下面测试)
}
```

```
测试1: darwin amd64
prints: 8

测试2: windows 386
prints: 4
```

### **<a name="not-cap-map">d. 不要cap map</a>**

```go
package main

func main() {
    var m = make(map[int]int, 10)
    println(cap(m))
}
```

编译错误:

```
# command-line-arguments
./test.go:5: invalid argument m (type map[int]int) for cap
```

### **<a name="for-range">e. for range的使用</a>**

**Valid usage:**

> slice:

```go
for range slice {}
```

```go
for i := range slice { _ = slice[i] }
```

```go
for i, v := range slice { _ = slice[i] == v }
```

```go
for _, v := range slice { _ = v }
```

> map:

```go
for range map {}
```

```go
for k := range map { _ = map[k] }
```

```go
for k, v := range map { _ = map[k] == v }
```

```go
for _, v := range map { _ = v }
```

> chan:

```go
for range chan {}
```

```go
for v := range chan { _ = v } // break after close(chan) and no more elem in chan
```

### **<a name="blank_var">f. ‘_’的使用</a>**

**Valid usage:**

```go
// 用于注册驱动等，运行init函数
import _ "github.com/go-sql-driver/mysql"

// 建议只用在测试
_ = v

// 用于获取值时
for _, v := range slice/map {}

// 不建议下面写法
for _ = range slice/map/chan {}
// 可替换为
for range slice/map/chan {}

// 字段名可以使用
// 1. 初始化时不可省略字段名
// 2. 为了内存对齐
struct { _ int }

// 建议只有当兼容接口并且参数无用时用 _
// 同样只有当兼容接口并且需要吞掉返回错误值时用 _
func(_ int) (_ error)
```

另外

```go
// 不建议写成下面形式:
func (_ T) funcName() {}

// 下划线在这没用，可以省略掉:
func (T) funcName() {}

// 因为参数T无用，更建议不要T:
func funcName() {}
```

### **<a name="short-var-dec">g. :=的使用</a>**

> a. 不能用于声明全局变量

```go
package main

a := 10

func main() {}
```

编译错误:

```
# command-line-arguments
./test.go:3: syntax error: non-declaration statement outside function body
```

> b. 不能用于结构体字段值

```go
package main

type foo struct {
    bar int
}

func main() {
    var f foo
    f.bar, tmp := 1, 2
}
```

编译错误:

```
# command-line-arguments
./test.go:9: non-name f.bar on left side of :=
```

> c. 隐藏变量作用域

```go
package main

func main() {
	var a  = 5
	if a, ok := 6, true; ok {
		println("a in if:", a) // prints: 6
	}
	println("a outer:", a) // prints: 5
}
```

**************************************************

**<a name="attention-string">String</a>**
=============================

### **<a name="special-rune">a. 注意特殊字符</a>**

> 一个字符可能占用多个runes.

```go
package main

import "unicode/utf8"

func main() {  
    data := "é"
    println(len(data))                    // prints: 3
    println(len([]rune(data)))            // prints: 2
    println(utf8.RuneCountInString(data)) // prints: 2
}
```

### **<a name="range-rune">b. range string尝试解析为rune</a>**

```go
package main

import "fmt"

func main() {
    data := "A\xfe\x02\xff\x04"
    for _,v := range data {
        fmt.Printf("%#x ",v)
    }
    fmt.Println()
    // prints: 0x41 0xfffd 0x2 0xfffd 0x4

    for _,v := range []byte(data) {
        fmt.Printf("%#x ",v)
    }
    fmt.Println()
    // prints: 0x41 0xfe 0x2 0xff 0x4
}
```

```go
package main

import "fmt"

func main() {
	for i, r := range "你好" {
		fmt.Printf("%d: %c\n", i, r)
	}
}
// Output: 注意下标
// 0: 你
// 3: 好
```

### **<a name="bytes-append-string">c. []byte可以append string</a>**

```go
package main

import "fmt"

func main() {
	fmt.Printf("%s\n", append([]byte(nil), "Hello world"...))
	// prints: Hello world
}
```

### **<a name="runes-append-string">d. []rune不可以append string</a>**

```go
package main

import "fmt"

func main() {
	fmt.Printf("%s\n", append([]rune(nil), "Hello world"...))
}
```

编译错误:

```
# command-line-arguments
./test.go:6: cannot use "Hello world" (type string) as type []rune in append
```

***

**<a name="attention-slice">Slice</a>**
=============================

### **<a name="init-slice-with-sub">a. 下标初始化方式</a>**

```go
package main

// 有...为数组, 无...为切片
var ls = [...]int{
	1: 3,
	3: 6,
	6: 9,
}

func main() {
	println(len(ls)) // prints: 7
}
```

### **<a name="when-ralloc">b. 什么时候会重新分配</a>**

```go
package main

// 机器不同, 地址可能不同
func main() {
    var slice []int
    println(slice) // prints: [0/0]0x0, 初始地址必然相同

    slice = make([]int, 0, 1) // make时重新分配内存
    println(slice) // prints: [0/1]0xc42003bf28

    slice = append(slice, 1) // cap足够时不重新分配
    println(slice) // prints: [1/1]0xc42003bf28

    slice = append(slice, 2) // cap不够时重新分配
    println(slice) // prints: [2/2]0xc42000a120
}
```

**注意: `切片修改, 涉及len和cap的变动, 要么用指针, 要么返回新切片`(建议返回切片)**

```go
package main

import "fmt"

func usePointer(ls *[]int) {
	*ls = append(*ls, 1)
}

func retSlice(ls []int) []int {
	return append(ls, 2)
}

func main() {
	var ls []int
	usePointer(&ls)
	fmt.Println(ls) // prints: [1]

	fmt.Println(retSlice(nil)) // prints: [2]
}
```

### **<a name="when-modify-value">c. 什么时候会改变值</a>**

> Slice in struct

```go
package main

import "fmt"

type T struct {
    ls []int
}

func foo(t T) { // t为传值
    t.ls[0] = 999 // 但字段ls却相当于变相传址
}

func main() {
    var t = T{ls: []int{1, 2, 3}}
    fmt.Println(t) // prints: {[1 2 3]}

    foo(t)
    fmt.Println(t) // prints: {[999 2 3]}
}
```

> Slice insert

```go
package main

import "fmt"

func main() {
    var ls = []int{0, 1, 3, 4}
    ls = append(append(ls[:2], 2), ls[2:]...)
    fmt.Println(ls)
    // prints: [0 1 2 2 4] but we want [0 1 2 3 4]
}
```

**正确写法: 用切片的完整写法** `slice[start:len:cap]`

```go
package main

import "fmt"

func main() {
    var ls = []int{0, 1, 3, 4}
    ls = append(append(ls[:2:2], 2), ls[2:]...)
    fmt.Println(ls)
    // prints: [0 1 2 3 4]
}
```

### **<a name="adjust-len-cap">d. 调整len和cap值</a>**

**本作法为奇技淫巧, 不推荐**

```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	var ls = []int{0, 1, 2, 3, 4}[:0]
	println(ls) // prints: [0/5]0xc4200740c0
	fmt.Println(ls) // prints: [] // 切片过后不知道原数组里的值

	var header = (*reflect.SliceHeader)(unsafe.Pointer(&ls))
	header.Len = header.Cap

	println(ls) // prints: [5/5]0xc4200740c0
	fmt.Println(ls) // prints: [0 1 2 3 4]
}
```

**************************************************

**<a name="attention-map">Map</a>**
=============================

### **<a name="map-with-ok">a. 什么时候不能省略ok</a>**

> **主要用于判断存在关系**<br/>
> 1. 当value是struct时，根据ok判断struct是不是空值<br/>
> 2. 判断value值是否存在时，最好用ok；如果value可能存在默认值时，只能用ok

```go
package main

func main() {
	var m = map[int]string{1: "asd", 2: "qwe"}
	if m[1] != "" { // 没有默认值存在时可以与默认值比较, 但最好还是用ok
		println("1 exist")
	}

	m[0] = "" // map中存在默认值
	if _, ok := m[0]; ok { // 这里只能用ok
		println("0 exist")
	}
}
```

### **<a name="map-without-ok">b. 什么时候最好省略ok</a>**

> **主要用于使用取出来的值时**<br/>
> 1. 当value是指针、map、chan、func时，据[永远不要省略判空操作](#must-check-nil)，取出来的指针是要判空的，所以无谓判ok<br/>
> 2. 当value是数字类型、字符串、切片、数组时，因为都可以直接用默认值，所以不用判ok

```go
package main

func main() {
	var m = make(map[rune]int)

	for _, r := range "Hello wrold" {
		m[r]++
	}

	println("'l' appears", m['l'], "times.")
}
```

### **<a name="map-to-set">c. 当Set(集合)用</a>**

> 这里，我们不关心value的值，只关心key的值时，即可以把map当set用，有两种形式:<br>
> 1. `map[key]bool`，用这种形式时，多是用于判断键值的存在与否，方便写if表达式<br/>
> 2. `map[key]struct{}`用这种形式时，多是只关心集合内有什么，多用于排重

**上述两种写法较之随手写的`map[key]int`之类有两个好处:**
> 1. 看到这两种写法时，让阅读代码者知道，去关心key，而不是value及key value间的对应关系。随手写的int或其他类型有一定误导作用；(主要好处，写代码的人要多敲几个字符，但看代码的人会减少心智负担)
> 2. 节约内存。(不是主要好处，因为量少的时候也节约不了多少)

```go
package main

func main() {
	var m = make(map[rune]struct{})

	for _, r := range "Hello wrold" {
		m[r] = struct{}{}
	}

	println("Total", len(m), "diffent characters.")
}
```

### **<a name="not-multi-op-safe">d. 非内存安全类型</a>**

> 1. 在读map之前可以不分配内存，nil map是可读的;
> 2. 在写map之前一定记得分配内存，nil map would panic；
> 3. 多协程并发读map时可以不加锁，但并发读发写时一定要记得加锁，否则panic

```go
package main

func main() {
	var m map[string]int
	println(len(m), m["abc"]) // prints: 0 0
}
```

```go
package main

func main() {
	var m = make(map[int]int)

	go func() {
		for i := 10; i != 0; i++ {
			m[i] = i
		}
	}()

	for i := 1; i != 0; i++ {
		m[i] = i * 10
	}
}
```

***panic:***

```
fatal error: concurrent map writes

goroutine 17 [running]:
runtime.throw(0x679bb, 0x15)
	/usr/local/go/src/runtime/panic.go:566 +0x95 fp=0xc420020688 sp=0xc420020668
runtime.mapassign1(0x59220, 0xc42006a000, 0xc4200207a0, 0xc420020798)
	/usr/local/go/src/runtime/hashmap.go:458 +0x8ef fp=0xc420020770 sp=0xc420020688
main.main.func1(0xc42006a000)
	/Users/sg/test/test.go:8 +0x66 fp=0xc4200207b8 sp=0xc420020770
runtime.goexit()
	/usr/local/go/src/runtime/asm_amd64.s:2086 +0x1 fp=0xc4200207c0 sp=0xc4200207b8
created by main.main
	/Users/sg/test/test.go:10 +0x73

goroutine 1 [runnable]:
main.main()
	/Users/sg/test/test.go:13 +0xc3
exit status 2
```

### **<a name="map-simplified-candy">e. 字面量简化写法</a>**

> 可以省略类型声明(Go1.5添加)

```go
package main

type Pos struct {
	x, y int
}

var m = map[Pos]map[string][]Pos{
	{0, 1}: {"hell": {{0, 1}, {1, 2}}},
	{3, 4}: {"heaven": {{3, 4}, {4, 5}}},
}

func main() {}
```

**************************************************


**<a name="attention-chan">Chan</a>**
=============================

### **<a name="chan-with-ok">a. 永远不要省略ok</a>**

```go
package main

func main() {
	var ch = make(chan int, 1)
	ch <- 1
	close(ch)

	for {
		v := <-ch
		println(v) // prints: 1, 0, 0, 0, 0............
	}
}
```

正确写法1:

```go
for v := range ch {
	println(v)
}
```

正确写法2:

```go
for {
	v, ok := <-ch
	if !ok {
		break
	}
	println(v)
}
```

### **<a name="closed-chan">b. closed chan</a>**

> 如上所述，可以不停地从closed chan中读取数据，所以closed chan一般用作协程同步

```go
package main

import "sync"

func main() {
	var ch = make(chan struct{})
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(n int) {
            <-ch // 等待close(ch)，所有协程同时开始
            println(n)
            wg.Done()
        }(i)
    }

	close(ch)
    wg.Wait()
}
// prints: 1 7 2 8 6 5 3 0 9 4 (随机的)
```

或者如:

```go
package main

func main() {
	var ch = make(chan struct{})

    go func() {
        println("Do something")
	    close(ch)
    }()

    <-ch
    println("Done")
}
```

> closed chan不可以再次被close

```go
package main

func main() {
	var ch = make(chan struct{})
	close(ch)
	close(ch)
}
```

***panic:***

```
panic: close of closed channel

goroutine 1 [running]:
panic(0x59640, 0xc42000a130)
	/usr/local/go/src/runtime/panic.go:500 +0x1a1
main.main()
	/Users/sg/test/test.go:6 +0x57
exit status 2
```

> chan关闭原则: 在写端关闭，在读端判断。

### **<a name="nil-chan">c. nil chan</a>**

> nil chan的读和写都会阻塞:

```go
package main

func main() {
	var ch chan int
	select {
	case v, ok := <-ch:
		println(v, ok)
	default:
		println("default") // prints: default
	}
}
```

> nil chan不可以被close

```go
package main

func main() {
    var ch chan int
    close(ch)
}
```

***panic:***

```
panic: close of nil channel

goroutine 1 [running]:
panic(0x59540, 0xc42000a130)
	/usr/local/go/src/runtime/panic.go:500 +0x1a1
main.main()
	/Users/sg/test/test.go:5 +0x2a
exit status 2
```

### **<a name="chan-with-after">d. 与time.After共用</a>**


```go
package main

import "time"

func main() {
    var ch = make(chan int, 5)
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)

    for {
        select {
        case v, ok := <-ch:
            if !ok { return }
            println("recv from chan:", v)
        case now := <-time.After(0):
            println("recv from time:", now.Nanosecond())
        }
    }
}

// Output:
// recv from chan: 0
// recv from chan: 1
// recv from chan: 2
// recv from chan: 3
// recv from chan: 4
```

> 如何能让time.After上位?

```go
package main

import "time"

func main() {
	var ch = make(chan int, 5)
	for i := 0; i < 5; i++ {
		ch <- i
	}
	close(ch)

	for {
		after := time.After(0) // time.Millisecond也可以
		time.Sleep(time.Millisecond)

		select {
		case v, ok := <-ch:
			if !ok {
				return
			}
			println("recv from chan:", v)
		case now := <-after:
			println("recv from time:", now.Nanosecond())
		}
	}
}

// Output:
// recv from time: 796825292
// recv from chan: 0
// recv from time: 799508258
// recv from time: 800867970
// recv from chan: 1
// recv from time: 803666609
// recv from chan: 2
// recv from chan: 3
// recv from time: 807868034
// recv from time: 809129591
// recv from time: 810508449
// recv from chan: 4
// recv from time: 813287164
```

**每次一个新的time.After()都是从新计算时间的，而且还要调用runtime中的函数，会有一定的时延，造成time.After(0)也不能上位。**

> 如果是定时任务，建议用time.Ticker

```go
package main

import "time"

func main() {
	var ch = make(chan int, 10)

    go func() {
        for i := 0; i < 10; i++ {
            ch <- i
            time.Sleep(time.Millisecond)
        }
        close(ch)
    }()

    var ticker = time.NewTicker(2 * time.Millisecond)
    defer ticker.Stop() // 记得Stop()

	for {
		select {
		case v, ok := <-ch:
			if !ok {
				return
			}
			println("recv from chan:", v)
		case now := <-ticker.C:
			println("recv from time:", now.Nanosecond())
		}
	}
}

// Output:
// recv from chan: 0
// recv from chan: 1
// recv from time: 864627593
// recv from chan: 2
// recv from chan: 3
// recv from time: 866541281
// recv from chan: 4
// recv from chan: 5
// recv from time: 868498022
// recv from chan: 6
// recv from time: 870687259
// recv from chan: 7
// recv from chan: 8
// recv from time: 872546370
// recv from chan: 9
// recv from time: 874715515
```

**************************************************

**<a name="attention-break">Break</a>**
=============================

### **<a name="break-switch">a. break switch</a>**

> break只能跳出switch，却跳不出for，程序永远不会结束

```go
package main

func main() {
    for {
        switch {
        default:
            break
        }
    }
}
```

### **<a name="break-select">b. break select</a>**

> break只能跳出select，却跳不出for，程序永远不会结束

```go
package main

func main() {
    for {
        select {
        default:
            break
        }
    }
}
```

### **<a name="to-single-func">c. 最好单独成函数</a>**

> 当然，一种解决办法就是break label

```go
package main

func main() {
LOOP:
    for {
        switch {
        default:
            break LOOP
        }
    }

    println("Done") // prints: Done
}
```

> 比较推荐的办法是，将switch或select抽成一个函数，不要和for搞在一起

```go
package main

func breakfor(i int) bool {
	switch {
	case i > 5:
		return true
	}
	return false
}

func main() {
	for i := 0; i < 10; i++ {
		if breakfor(i) {
			println("break for on i is", i)
			break
		}
	}
}
// prints: break for on i is 6
```

### **<a name="goto-beyond-func">d. goto不能跨函数跳</a>**

```go
package main

func main() {
	func() {
		goto END
	}()
END:
}
```

```
# command-line-arguments
./test.go:5: label END not defined
./test.go:7: label END defined and not used
```

**************************************************

**<a name="attention-ptr">Pointer</a>**
=============================

### **<a name="must-check-nil">a. 永远不要省略判空操作</a>**

> 在项目中，建议永远不要省略对指针的判空操作，因为指不定有哪些人会怎么调这个函数

```go
package main

type T struct {
	someSegment interface{}
}

func process(t *T) {
	if t == nil {
		// do some log or others
		return
	}

	// do something with t...
}

func main() {
	process(nil)
}
```

> 如果有必要的话，方法的接收者也要判空

```go
package main

import "fmt"

type T struct {
	someSegment interface{}
}

func (t *T) String() string {
	if t == nil {
		return "<nil>"
	}

	return fmt.Sprint(t.someSegment)
}

func main() {
	fmt.Println((*T)(nil)) // prints: <nil>
}
```

**同理的还有map, chan, func，总之一切有可能出错的地方，尽量加上判空操作。**

**如果是包内信任，可以适当省略判空操作。**

### **<a name="mem-align-addr">b. 不可寻址的情况</a>**

> map[key]struct不可寻到struct的物理地址

```go
package main

type T struct {
	n int
}

func main() {
	var m = make(map[int]T)

	m[0].n = 1
}
```

编译错误:

```
# command-line-arguments
./test.go:10: cannot assign to struct field m[0].n in map
```

> 返回struct不可寻到struct的物理地址

```go
package main

type T struct {
	n int
}

func getT() T {
	return T{}
}

func main() {
	getT().n = 1
}
```

编译错误:

```
# command-line-arguments
./test.go:12: cannot assign to getT().n
```

> **不可寻址的结构不能调用带结构体指针接收者的函数**

```go
package main

type T struct {
	n int
}

func (t *T) Set(n int) {
	t.n = n
}

func getT() T {
	return T{}
}

func main() {
	getT().Set(1)
}
```

编译错误:

```
# command-line-arguments
./test.go:16: cannot call pointer method on getT()
./test.go:16: cannot take the address of getT()
```

> **解决办法: 用指针，或用一临时变量**

```go
package main

type T struct {
	n int
}

func (t *T) Set(n int) {
	t.n = n
}

func getT() T {
	return T{}
}

func main() {
	var m = map[int]*T{1: &T{}}
	m[1].n = 1 // 一定要先判断存在与否，否则会nil pointer dereference

	var t = getT()
	t.Set(2)
}
```

**************************************************

**<a name="attention-func">Func</a>**
=============================

### **<a name="func-cmp-nil">a. 只能与nil比较</a>**

> valid:

```go
package main

func main() {
	var fn = func() {}

	if fn != nil {
		println("fn is not nil")
	}
}
```

> invalid:

```go
package main

func main() {
	var fn1 = func() {}
	var fn2 = func() {}

	if fn1 != fn2 {
		println("fn1 not equal fn2")
	}
}
```

编译错误:

```
# command-line-arguments
./test.go:7: invalid operation: fn1 != fn2 (func can only be compared to nil)
```

### **<a name="func-closure">b. 闭包使用</a>**

> 1). 最常见的格式: go func()，注意公共变量

错误格式(1):

```go
package main

import "time"

func main() {
	for i := 0; i < 5; i++ {
		go func() {
			println(i) // prints: 5 5 5 5 5
		}()
	}

	time.Sleep(100 * time.Millisecond)
}
```

错误格式(2):

```go
package main

import "time"

func main() {
	for i := 0; i < 5; i++ {
		go func(n *int) {
			println(*n) // prints: 5 5 5 5 5
		}(&i)
	}

	time.Sleep(100 * time.Millisecond)
}
```

**正确格式(1):**

```go
package main

import "time"

func main() {
	for i := 0; i < 5; i++ {
		i := i
		go func() {
			println(i) // prints: 0 2 3 4 1
		}()
	}

	time.Sleep(100 * time.Millisecond)
}
```

**正确格式(2):**

```go
package main

import "time"

func main() {
	for i := 0; i < 5; i++ {
		go func(n int) {
			println(n) // prints: 2 4 1 0 3
		}(i)
	}

	time.Sleep(100 * time.Millisecond)
}
```

**range的情况相同，这里不做赘述**

> 2). 递归

```go
package main

func main() {
	var fn = func(n int) {
		if n < 0 {
			return
		}
		println(n)
		fn(n - 1)
	}

	fn(5)
}
```

编译错误:

```
# command-line-arguments
./test.go:9: undefined: fn
```

**正确写法:**

```go
package main

func main() {
	var fn func(int)
	fn = func(n int) {
		if n < 0 {
			return
		}
		println(n)
		fn(n - 1)
	}

	fn(5)
}
// prints: 5 4 3 2 1 0
```

> 3). for循环defer

```go
package main

import "sync"

func main() {
	var mtx sync.Mutex
	var m = make(map[int]int)

	for i := 0; i < 5; i++ {
		mtx.Lock()
		defer mtx.Unlock()
		m[i] = i << 8 // 或是一些其他什么操作
	}
}
```

本想是离开scope时做的事情，结果出现意外的结果

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0xc42000a0ac)
	/usr/local/go/src/runtime/sema.go:47 +0x30
sync.(*Mutex).Lock(0xc42000a0a8)
	/usr/local/go/src/sync/mutex.go:85 +0xd0
main.main()
	/Users/sg/test/test.go:10 +0xf3
exit status 2
```

**正确写法:**

```go
package main

import "sync"

func main() {
	var mtx sync.Mutex
	var m = make(map[int]int)

	for i := 0; i < 5; i++ {
		func() {
			mtx.Lock()
			defer mtx.Unlock()
			m[i] = i << 8
		}()
	}
}
```

### **<a name="func-with-receiver">c. 带有接收者的函数</a>**

> 带有接收者函数的三种调用方式:

```go
package main

type T struct {
	s string
}

func (t *T) greet() {
	println(t.s)
}

func main() {
	var t = &T{"Hello"}
	t.greet()

	var greet = (*T).greet
	greet(&T{"你好"})

	var greeting = (&T{"こんにちは"}).greet
	greeting()
}
```

### **<a name="func-go-defer">d. 搭配go、defer时代码的执行顺序</a>**

> go、defer关键字新开协程，只针对最终要调用的函数，参数中如果有函数，不是在新协程里或原函数退出时执行的

```go
package main

import "time"

func Defer() (_ string) {
	println("defer")
	return
}

func Go() (_ string) {
	println("go")
	return
}

func main() {
	defer print(Defer())
	go print(Go())
	println("println")
	time.Sleep(10 * time.Millisecond)
}
// Output:
// defer
// go
// println
```

如果想要类似println, go, defer这种顺序的结果，应该用func(){}把要执行的语句块包起来:

```go
defer func() {
	print(Defer())
}()

go func() {
	print(Go())
}()
```

### **<a name="func-for-range">e. 再续for range组合</a>**

> 期望输出1, 2, 3的各种组合吗？Not in your life!

```go
package main

import "time"

type T struct {
	n int
}

func (t *T) print() {
	println(t.n)
}

func main() {
	var ts = []T{{1}, {2}, {3}}
	for _, t := range ts {
		go t.print()
	}
	time.Sleep(10 * time.Millisecond)
}

// prints: 3 3 3
```

它和下面代码等价:

```go
func main() {
	var ts = []T{{1}, {2}, {3}}
	var printT = (*T).print
	var t T
	for i := range ts {
		t = ts[i]
		go printT(&t)
	}
	time.Sleep(10 * time.Millisecond)
}
```

**注意，参考[结构体函数](#func-of-struct)的第二种写法，t.print()是要取t的地址传过去，上面的写法最明显的漏洞便在于t的地址没变，而t的值却连变3次**

**解决办法也很简单:**

**办法1:**

```go
var ts = []*T{{1}, {2}, {3}} // 变成the slice of pointers
```

**办法2:**

```go
var ts = []T{{1}, {2}, {3}}
for i := range ts {
	go ts[i].print()
}
```

**其它办法参见[闭包使用](#func-closure)。**

**map的情况相同，这里不做赘述**

**************************************************

**<a name="attention-interface">Interface</a>**
=============================

### **<a name="nil-interface">a. nil interface</a>**

> 只有当直接给interface{}赋值为nil时，interface{}才等于nil:

```go
package main

func main() {
	var i interface{} = nil
	println(i == nil) // prints: true

	i = (*int)(nil)
	println(i == nil) // prints: false
}
```

> interface{}为nil的条件:<br/>
> 1. interface{}由两部分组成: type和value，只要其中有一不为nil，则interface{}不为nil<br/>
> 2. 不可能出现type为nil而value不为nil的情况<br/>
> 由上可得出判别interface{}是否为nil的简便办法: %T

```go
package main

import "fmt"

func main() {
	var i interface{} = nil
	fmt.Printf("i == nil: %v, type: %T\n", i == nil, i)
	// prints: i == nil: true, type: <nil>

	i = (*int)(nil)
	fmt.Printf("i == nil: %v, type: %T\n", i == nil, i)
	// prints: i == nil: false, type: *int
}
```

```go
package main

func isNil(i interface{}) bool {
	switch i.(type) {
	case nil:
		return true
	}
	return false
}

func main() {
	println(isNil(nil)) // prints: true

	var t *int
	var i interface{} = t
	println(isNil(t)) // prints: false

	println(i == (*int)(nil)) // prints: true
}
```

### **<a name="type-assert">b. 类型断言</a>**

> 建议使用类型断言的时候永远不要省略`ok`，否则如果类型不一致会panic

```go
package main

func main() {
	var i interface{} = 5
	n, ok := i.(float64)
	if !ok {
		println("i is not float64")
		return
	}
	println(n)
}
// prints: i is not float64
```

```go
package main

func main() {
	var i interface{} = 5
	println(i.(float64))
}
```

panic:

```
panic: interface conversion: interface is int, not float64

goroutine 1 [running]:
panic(0x59960, 0xc42006a000)
	/usr/local/go/src/runtime/panic.go:500 +0x1a1
main.main()
	/Users/sg/test/test.go:5 +0x8e
exit status 2
```

**同样，包内信任时可省略`ok`判断:**

```go
package main

import "container/list"

type IntList struct {
	l *list.List // l字段并未向外导出, 外部不可见
}

func New() IntList { // 注意并没有返回指针, 想想和上面slice中的共同特点
	return IntList{l: list.New()}
}

func (ls IntList) Push(v int) {
	ls.l.PushBack(v)
}

func (ls IntList) Pop() int {
	elem := ls.l.Front()
	if elem == nil {
		return 0 // empty
	}
	ls.l.Remove(elem)
	return elem.Value.(int) // 结构体内只可能是int
}

func main() {
	ls := New()
	ls.Push(1)
	ls.Push(2)
	println(ls.Pop()) // prints: 1
	println(ls.Pop()) // prints: 2
	println(ls.Pop()) // prints: 0
}
```

### **<a name="implement-interface">c. 实现接口的要求</a>**

> 由于在interface{}内部不可以对变量进行寻址，所以下面程序是错误的:

```go
package main

import "io"

type NopReader struct{}

func (*NopReader) Read(b []byte) (int, error) {
    return len(b), nil
}

func main() {
    var _ io.Reader = NopReader{}
}
```

编译错误:

```
# command-line-arguments
./test.go:12: cannot use NopReader literal (type NopReader) as type io.Reader in assignment:
	NopReader does not implement io.Reader (Read method has pointer receiver)
```

> 而在interface{}内是可以进行解指针引用操作的，所以下面程序是合法的:

```go
package main

import "io"

type NopReader struct{}

func (NopReader) Read(b []byte) (int, error) {
	return len(b), nil
}

func main() {
	var _ io.Reader = &NopReader{}
}
```

**注意: 在Go1.4后不再在`**T`进行解引用，也就意味着下面程序是错的**

```go
package main

import "io"

type NopReader struct{}

// 无论此处是不是指针
func (*NopReader) Read(b []byte) (int, error) {
	return len(b), nil
}

func main() {
	var nop = &NopReader{}
	var _ io.Reader = &nop
}
```

编译错误:

```
# command-line-arguments
./test.go:13: cannot use &nop (type **NopReader) as type io.Reader in assignment:
	**NopReader does not implement io.Reader (missing Read method)
```

**************************************************

**<a name="attention-comment">Comment</a>**
=============================

### **<a name="build-tag-comment">a. build tag</a>**

```
	./
	|- mypkg/
	|    +- win.go
	+- test.go
```
> win.go

```go
// +build windows

package mypkg

func init() {
    println("This file will only be compiled on windows.")
}
```

> test.go

```go
package main

import _ "./mypkg"

func main() {}
```

> 编译错误:

```
test.go:3:8: no buildable Go source files in /Users/sg/test/mypkg
```

> 将 `+build windows` 改成 `+build !windows`, 输出:

```
This file will only be compiled on windows.
```

> 编译规则:<br/>
> 1). 编译标签由空格分隔的编译选项(options)以"或"的逻辑关系组成<br/>
> 2). 每个编译选项由逗号分隔的条件项以逻辑"与"的关系组成<br/>
> 3). 每个条件项的名字用字母+数字表示，在前面加!表示否定的意思<br/>
> 4). 多个编译标签之间是逻辑"与"的关系

### **<a name="package-tag-comment">b. package tag</a>**

```
	./
	|- src/
	|   +- mypkg/
	|        +- my.go
	+- test.go

	export GOPATH=`pwd`
```

> my.go

```go
package mypkg // import "third/mypkg"

func init() {
	println(`This pkg must import with "third/mypkg"`)
}
```

> test.go

```go
package main

import _ "mypkg"

func main() {}
```

> 编译错误:

```
test.go:3:8: code in directory /Users/sg/test/src/mypkg expects import "third/mypkg"
```

**不建议使用pkg tag来表明包的导入方式，否则当包的路径变化后，会出现如上，导入失败的情况**

*更多请参见: [Go's Hidden Pragmas](https://github.com/gopherchina/conference/blob/master/2017/2.4%20Go's%20Hidden%20Pragmas.pdf)*

**************************************************

**<a name="vendor">Vendor</a>**
=============================

### **<a name="diff-pkg">a. vendor中的包和外面的包是两个不同的包</a>**

**以下目录:**

```
  ./
  |- test.go
  +- src/
      |- mypkg/
      |    |- my.go
      +- vender/
           |- vd.go
           +- vendor/
                +- mypkg/
                     +- my.go
```

```go
// test.go
package main

import (
  _ "mypkg"
  _ "vender"
)

func main() {
}
```

```go
// mypkg/my.go 
package mypkg

func init() {
  println("init mypkg")
}
```

```go
// vender/vd.go
package vender

import _ "mypkg"
```

```go
// vender/vendor/mypkg/my.go 
package mypkg

func init() {
  println("init mypkg in vendor")
}
```

运行`test.go`:

```
init mypkg
init mypkg in vendor
```

> 有关vendor的详细说明，可以参考[GopherCon 2016: Wisdom Omuya - Go Vendoring](https://www.youtube.com/watch?v=6gdVhHMxNTo)(感谢丹丹哥Daniel提供)

***当用官方包`database/sql`时须注意: 每个驱动只能注册一次，否则会panic，如果在导入的包中嵌有vendor，记得检查会不会重复注册驱动！！！***

**************************************************

**<a name="bit-op">Bit operation</a>**
=============================

### **<a name="diff-priority">a. 优先级与其他语言不同</a>**

先看例子:

```c
#include <stdio.h>

int main() {
    printf("%d\n", 1 << 1 + 1); // prints: 4
    printf("%d\n", 6 & 4 * 3);  // prints: 4
    return 0;
}
```

```go
package main

func main() {
    println(1 << 1 + 1) // prints 3, not 4
    println(6 & 4 * 3)  // prints 12, not 4
}
```

> golang的运算符优先级

| 优先级 | 运算符            |
|:----: |:---------------- |
|   7   | ^ !              |
|   6   | * / % << >> & &^ |
|   5   | + - \| ^         |
|   4   | == != < <= >= >  |
|   3   | <-               |
|   2   | &&               |
|   1   | \|\|             |

**注意，如果不是特别清楚，最好加上`()`以保证运算优先级**

### **<a name="diff-flip">b. 取反操作符与其他语言不同</a>**

> 取反操作，在c语言及其他很多语言中是`~`，在go语言中是`^`

```go
// go
package main

func main() {
    println(^9) // prints: -10
}
```

```python
# python
print ~9 # prints: -10
```

```c
// c
#include <stdio.h>

int main() {
    printf("%d\n", ~9); // prints: -10
    return 0;
}
```

### **<a name="less-bit-op">c. 不是写算法尽量少用位运算</a>**

**注意: `位运算本身不容易理解, 不是写算法要求效率时, 尽可能少用位运算`**

> 举例

```
package main

import "fmt"

// 来源: C语言
func setZero() {
	var t = 5
	println(t ^ t)  // prints: 0
	println(t &^ t) // prints: 0
}

// 来源: C语言
func abs() {
	var x int32 = -9
	var y int32 = x >> 31
	println((x ^ y) - y) // prints: 9
}

// 来源: C语言
func swap() {
	var x, y = 5, 6
	x ^= y
	y ^= x
	x ^= y
	println(x, y) // prints: 6 5
}

// 来源: C语言, 防溢出
func average() {
	var x, y = 5, 7
	println((x & y) + ((x ^ y) >> 1)) // prints: 6
}

// 来源: 《汇编语言(第2版)》 --王爽
func transLetter() {
	fmt.Printf("%c\n", 'A'|32)        // prints: a
	fmt.Printf("%c\n", 'a'&^byte(32)) // prints: A

	fmt.Printf("%c\n", 'A'^32) // prints: a
	fmt.Printf("%c\n", 'a'^32) // prints: A
}

// 来源: 数状数组.lowbit()
// 《编程之美——微软技术面试心得》2.1 求二进制数中1的个数
func lastBit1() {
	var t = 12
	println(t & -t)            // prints: 4
	println(t & (t ^ (t - 1))) // prints: 4
	println(t ^ (t & (t - 1))) // prints: 4
	println(t - (t & (t - 1))) // prints: 4
}

func main() {
	setZero()
	abs()
	swap()
	average()
	transLetter()
	lastBit1()
}
```

另外还有很多常用位运算如<<、>>等，还是那句老话，***不写算法，少用位运算，不好理解。***

**************************************************

**<a name="pkg-encoding-json">pkg encoding/json</a>**
=============================

### **<a name="json-set-escape-HTML">a. SetEscapeHTML(感谢Jason Qu)</a>**

> Golang在序列化字符串json时，会将'<'、'>'、'&'转换成对应的unicode值:

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	js, _ := json.Marshal("<>&")
	fmt.Printf("%s\n", js)
	// prints: "\u003c\u003e\u0026"
}
```

> Go1.7后可以通过设置`SetEscapeHTML(false)`使json编码不对上述三个字符做序列化处理:

```go
package main

import (
	"encoding/json"
	"os"
)

func main() {
	encoder := json.NewEncoder(os.Stdout)
	encoder.SetEscapeHTML(false)
	encoder.Encode("<>&")
	// prints: "<>&"
}
```

### **<a name="json-map-int">b. map[int]...</a>**

> 自Go1.7开始，json序列化支持`map[int]...`类型(int, uint, int32...)的序列化与反序列化:

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	js, _ := json.Marshal(map[int]string{1: "a"})
	fmt.Printf("%s\n", js) // prints: {"1":"a"}

	var m map[int]string
	json.Unmarshal(js, &m)
	fmt.Println(m) // prints: map[1:a]
}
```

### **<a name="json-marshal-indent">c. MarshalIndent</a>**

> 格式化json字符串:

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	js, _ := json.MarshalIndent(map[int]string{1: "a", 2: "b"}, "", "  ")
	fmt.Printf("%s\n", js)
}
```

Output:

```
{
  "1": "a",
  "2": "b"
}
```
### **<a name="json-buffered">d. Buffered </a>**

> json.Decoder在反序列化过程中会buffer一部分数据，如果读到的数据不全是json，在后续的反序列化的过程中会出错

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io/ioutil"
)

func main() {
	buffer := bytes.NewBuffer([]byte(`{"1":"a"}0123456789`))
	decoder := json.NewDecoder(buffer)

	var m map[int]string
	decoder.Decode(&m)
	fmt.Println(m)
	// prints: map[1:a]

	fmt.Printf("remain: %q\n", buffer.Bytes())
	// prints: remain: ""

	bs, _ := ioutil.ReadAll(decoder.Buffered())
	fmt.Printf("buffered: %q\n", bs)
	// prints: buffered: "0123456789"
}
```

**************************************************

**<a name="pkg-container">pkg Container</a>**
=============================

### **<a name="container-list-remove">a. List.Remove(感谢Seashore)</a>**

```go
package main

import (
	"container/list"
)

func main() {
	ls := list.New()
	ls.PushBack(1)
	ls.PushBack(2)
	for e := ls.Front(); e != nil; e = e.Next() {
		ls.Remove(e)
	}
	println(ls.Len()) // prints: 1
}
```

**See: $GOROOT/src/container/list/list.go +108**

```go
// remove removes e from its list, decrements l.len, and returns e.
func (l *List) remove(e *Element) *Element {
    e.prev.next = e.next
    e.next.prev = e.prev
    e.next = nil // avoid memory leaks
    e.prev = nil // avoid memory leaks
    e.list = nil 
    l.len--
    return e
}
```

解决办法:

```go
	var next *list.Element
	for e := ls.Front(); e != nil; e = next {
		next = e.Next()
		ls.Remove(e)
	}
```

**************************************************

**<a name="pkg-unsafe">pkg unsafe (不推荐使用该包)</a>**
=============================

### **<a name="judge-endian">a. 判断机器大小端</a>**

> 简单写法:

```go
package main

import "unsafe"

func main() {
    var a uint32 = 1
    if *(*uint8)(unsafe.Pointer(&a)) == 1 {
        println("Little Endian")
    } else {
        println("Big Endian")
    }
}
```

> 看起来正规一些的写法:

```go
package main

import (
    "encoding/binary"
    "reflect"
    "unsafe"
)

func main() {
    var v uint32 = 1
    var b = *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
        Data: uintptr(unsafe.Pointer(&v)),
        Len:  binary.Size(v),
        Cap:  binary.Size(v),
    }))

    if binary.LittleEndian.Uint32(b) == v {
        println("Little Endian")
    } else {
        println("Big Endian")
    }
}
```

### <a name="changable-string">b. 可变字符串</a>

> 字符串是只读变量，这里用unsafe实现可写字符串

> 原理见: [Slice 调整len和cap值](#adjust-len-cap)

```go
package main

import (
	"fmt"
	"unsafe"
)

type ChangableString struct {
	b []byte
	s *string
}

func NewChangableString(s string) *ChangableString {
	var c = new(ChangableString)
	c.b = []byte(s)
	c.s = (*string)(unsafe.Pointer(&c.b))
	return c
}

func (c *ChangableString) Bytes() []byte {
	bs := make([]byte, len(c.b))
	copy(bs, c.b)
	return bs
}

func (c *ChangableString) String() string {
	return *c.s
}

func (c *ChangableString) At(i int) byte {
	if i < 0 || i > len(c.b) {
		return 0
	}

	return c.b[i]
}

func (c *ChangableString) Set(i int, b byte) {
	if i < 0 || i > len(c.b) {
		return
	}

	c.b[i] = b
}

func (c *ChangableString) Append(bs []byte) {
	c.b = append(c.b, bs...)
}

func (c *ChangableString) AppendStr(s string) {
	c.b = append(c.b, s...)
}

func (c *ChangableString) Cut(start, end int) {
	if start < 0 {
		return
	}
	if end > len(c.b) {
		return
	}
	if start > end {
		return
	}

	c.b = c.b[start:end]
}

func (c *ChangableString) Len() int {
	return len(c.b)
}

func main() {
	var s = NewChangableString(`123456789`)
	fmt.Println(s) // prints: 123456789

	s.Set(0, 'a')
	fmt.Println(s) // prints: a23456789

	s.AppendStr(`abcdefg`)
	fmt.Println(s) // prints: a23456789abcdefg

	s.Cut(9, s.Len())
	fmt.Println(s) // prints: abcdefg

	s.Cut(0, 3)
	fmt.Println(s) // prints: abc
}
```

**************************************************

