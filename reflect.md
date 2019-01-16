# Go语言反射

### 背景

先看官方Doc中Rob Pike给出的关于反射的定义：
Reflection in computing is the ability of a program to examine its own structure, particularly through types; it's a form of metaprogramming. It's also a great source of confusion.
(在计算机领域，反射是一种让程序——主要是通过类型——理解其自身结构的一种能力。它是元编程的组成之一，同时它也是一大引人困惑的难题。)

网上关于Golang反射的教程、博客，早已汗牛充栋。可遍数之下，有些偏于底层实现，有些偏于单一问题，有些偏于性能，有些是官方Doc的翻译，而且质量参差不齐。

在这种情况下，对于想学反射编程的人来说，很难看到一个完整的、系统的关于Golang反射编程的教程，甚至于在某段时间下好像学会了Golang反射，过段时间又忘了，好像从没学过一样。

本文的出现就是为了解决上述问题，争取对Golang反射编程，做一个尽量完善、系统的阐述。本文主要关注点在于：如何学会并使用Golang反射。因此不会讲述太多底层实现和性能方面的问题。

为了便于理解、记忆，本文在阐述过程中，会通过对比Golang一般语法和反射“语法”的方式，来说明如何使用反射编程。

最后提醒一句：有限制地用反射，即便你用的非常6。

### 基础
上面已经说到，反射主要通过类型，来理解其自身结构，所以反射中，比较关键的一点在于：类型。
另外，编程中所有的操作，都是对数据的操作，亦即值，所以反射中，另外一点在于：值。
在反射编程时，一定要时刻注意，你在操作变量的类型是什么，要对值做什么操作。
Golang反射中，类型的起点是reflect.TypeOf(x)，值的起点是reflect.ValueOf(x)。

#### 可读和可写
先看一个简单的例子(后面例子会省略package和import部分)：

```go
package main

import "reflect"

func main() {
	var i int = 5
	v := reflect.ValueOf(i)
	println(v.Int()) // Output: 5
}
```

一定要注意一点，`reflect.ValueOf(i)`是个函数调用，所以对比看一下下面这个没有反射的例子：

```go
func echo(v int) {
	println(v)
}

func main() {
	var i int = 5
	echo(i)
}
```

这两个例子是等价的，在上面反射中，`reflect.ValueOf(i)`是函数调用，函数调用分为传值和传址，该写法明显是在传值，它是`i`的一份拷贝，就像在调用`echo(i)`的时候，给`echo`传的是`i`的值，即在`echo`内部，`i`的值是只读的。
所以在下面例子中：

```go
func echo(v int) {
	v = 100
	println(v) // Output: 100
}

func main() {
	var i int = 5
	echo(i)
	println(i) // Output: 5
}
```

在`echo`中，将`v`的值改成了100，却改不了`main`中`i`的值，因为调用`echo`的本质是传值。传值的意思即只读，因此如果在反射中，对只读的值做任何`Set`操作，都会`panic`：

```go
func main() {
	var i int = 5
	v := reflect.ValueOf(i)
	v.SetInt(100) // panic
}
```

那如何改变值呢？很明显，应该传址：

```go
func change(v *int) {
	*v = 100
}

func main() {
	var i int = 5
	change(&i)
	println(i) // Output: 100
}
```

注意在该程序中:

```
1. 为了在函数`change`中改变`i`的值，我们将`i`的内存地址传给了`change`；

2. 在改变`i`的值时，`change`先给`v`做了解指针操作`*v`。
```

所以，对应的，反射也是函数调用，写出来，和上面逻辑上应该是一样的：

```go
func main() {
	var i int = 5
	v := reflect.ValueOf(&i)
	v.Elem().SetInt(100)
	println(i) // Output: 100
}
```

Golang反射简易之处在于：只要你学会了该语言，你就学会了反射。
比较一下上面两个程序，有什么不同吗？

- `v := reflect.ValueOf(&i)` 和 `var v *int = &i` 是一样的；
- `v.Elem().SetInt(100)` 和 `*v = 100` 是一样的，先解指针，再赋值。

### 如何理解
和非反射编程对比
CanInterface
CanSet
IsValid

### 如何使用
从循环说起，解决重复逻辑问题
什么时候用？不能用循环的情况下，比如struct.Fields，在语法上，不允许用循环。
(重点：忽略类型，逻辑相同)
不限，但请不要过度设计。

eg.
1. mysql

