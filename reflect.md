# Go语言反射

## 背景

先看官方Doc中Rob Pike给出的关于反射的定义：

> Reflection in computing is the ability of a program to examine its own structure, particularly through types; it's a form of metaprogramming. It's also a great source of confusion.
> (在计算机领域，反射是一种让程序——主要是通过类型——理解其自身结构的一种能力。它是元编程的组成之一，同时它也是一大引人困惑的难题。)

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

### 可读与可写
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

一定要注意一点，`reflect.ValueOf(i)`是个**函数调用**，所以对比看一下下面这个没有反射的例子：

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

那有没有方法能判断是否可写呢？`reflect.Value.CanSet() bool`该函数可以判断一个值是否可写：

```go
func main() {
    var i int = 5
    v := reflect.ValueOf(i)
    println(v.CanSet()) // Output: false
    v = reflect.ValueOf(&i)
    println(v.CanSet()) // Output: false // 标记1
    println(v.Elem().CanSet()) // Output: true
}
```

`标记1`处为什么是`false`呢？因为你要改的是指针指向的值，不是指针本身的值。

那具体有哪些是`CanSet`的呢？文档：

> A Value can be changed only if it is addressable and was not obtained by the use of unexported struct fields.

注意文档中的一句：`未被导出的struct fields是不可被Set的。`

### 取址

通过上面例子，我们知道，想要改变值，其前提是：取址。那什么样的变量是可取址的呢？反射包里有一个函数：`reflect.Value.CanAddr() bool`，该函数可以判断是否可取址。

```go
func main() {
    var i int = 5
    v := reflect.ValueOf(i)
    println(v.CanAddr()) // Output: false
    v = reflect.ValueOf(&i)
    println(v.CanAddr()) // Output: false
    println(v.Elem().CanAddr()) // Output: true
}
```

在该函数的文档中，已经明确了哪些可以被取址：

> A value is addressable if it is an element of a slice, an element of an addressable array, a field of an addressable struct, or the result of dereferencing a pointer. 

为方便理解，我将上述文档翻译成代码：

```go
func main() {
    ls := []int{1}
    println(reflect.ValueOf(ls).Index(0).CanAddr())         // Output: true
    println(reflect.ValueOf(&ls).Elem().Index(0).CanAddr()) // Output: true

    var t struct{ a, B int } 
    println(reflect.ValueOf(&t).Elem().Field(0).CanAddr()) // Output: true
    println(reflect.ValueOf(&t).Elem().Field(0).CanSet())  // Output: false
    println(reflect.ValueOf(&t).Elem().Field(1).CanAddr()) // Output: true
    println(reflect.ValueOf(&t).Elem().Field(1).CanSet())  // Output: true

    var p *int
    println(reflect.ValueOf(p).Elem().CanSet()) // Output: false
    p = &ls[0]
    println(reflect.ValueOf(p).Elem().CanSet()) // Output: true
}
```

既然知道了哪些可以取址，那么也就知道了哪些不可被取址，比如`map[k]v`中的`k`和`v`都不可被取址，没有传址进来的变量都不可被取址……

一定要注意：可取址`CanAddr`和可赋值`CanSet`不完全等价，主要区别在于`未被导出的struct fields`。

对指针和结构体相关的比较好理解，关键问题在于：为什么`an element of a slice`是可被取址和赋值的呢？

```go
func change(ls []int) {
    ls[0] = 100 
}

func main() {
    ls := []int{5}
    change(ls)
    println(ls[0]) // Output: 100
}
```

**在Golang里面，`slice`、`map`、`chan`默认是传址的！**
请看：

```go
func main() {
    ls := []int{0}
    reflect.ValueOf(ls).Index(0).SetInt(123)
    println(ls[0]) // Output: 123

    m := make(map[string]int)
    reflect.ValueOf(m).SetMapIndex(reflect.ValueOf("abc"), reflect.ValueOf(456))
    println(m["abc"]) // Output: 456

    ch := make(chan int, 1)
    reflect.ValueOf(ch).Send(reflect.ValueOf(789))
    println(<-ch) // Output: 789
}
```

**注意`slice`、`map`、`chan`默认传址的前提是：已经提前分配好内存空间！**

*细节知识点：`array`默认是传值。*

```go
func main() {
    arr := [...]int{1}
    println(reflect.ValueOf(arr).Index(0).CanAddr()) // Output: false
    println(reflect.ValueOf(arr).Index(0).CanSet())  // Output: false
}
```

### 类型

到目前为止，还没接触到类型。获取数据的类型可以用`reflect.ValueOf(x).Kind()`获取真实类型。所以有：

```go
func main() {
    var i int 
    println(reflect.ValueOf(i).Kind().String()) // Output: int

    var s string
    println(reflect.ValueOf(s).Kind().String()) // Output: string

    var iface interface{}
    println(reflect.ValueOf(iface).Kind().String())                // Output: invalid // 标记1
    println(reflect.ValueOf(&iface).Elem().Kind().String())        // Output: interface // 标记2
    println(reflect.ValueOf(&iface).Elem().Elem().Kind().String()) // Output: invalid // 标记3

    var t time.Time
    println(reflect.ValueOf(t).Kind().String()) // Output: struct // 标记4
}
```

**标记1、2、3处怎么理解？**
- 再次强调`reflect.ValueOf(x)`是**函数调用**；
- 注意函数签名：`func reflect.ValueOf(interface{}) reflect.Value`，参数是`interface{}`，指所有调用该函数的地方，都会将参数**类型转换**成`interface{}`；
- `interface{}`是`(typ, val)`对，空的`interface{}`是`(nil, nil)`，类型和值都是空，所以`Kind()`是`invalid`；
- 所有类型转换到`interface{}`的过程，都是`(typ, val)`拷贝的过程；
- 标记2处，`&iface`是一个`interface`类型的指针；
- 指针指向对象的类型是`interface`类型的变量；
- 该`interface`类型的变量的`Kind()`是`invalid`；
- 所以标记3处是`invalid`。

**标记4处，`time.Time`类型的底层是`struct`，即：`type Time struct { ... }`。**


那如果说想获得类型名，而不是底层类型，该怎么处理呢？`reflect.TypeOf(x)`：

```go
func main() {
    var i int 
    println(reflect.TypeOf(i).String()) // Output: int

    var s string
    println(reflect.TypeOf(s).String()) // Output: string

    var iface interface{}
    println(reflect.TypeOf(iface).String())         // panic
    println(reflect.TypeOf(&iface).String())        // Output: *interface {}
    println(reflect.TypeOf(&iface).Elem().String()) // Output: interface {}

    var t time.Time
    println(reflect.TypeOf(t).String()) // Output: time.Time
}
```

`reflect.TypeOf(x).String()`一般用来处理`time.Time`这种想要特殊处理的结构。

**构造常用类型：**

```go
func main() {
    intTyp := reflect.TypeOf(123)
    strTyp := reflect.TypeOf("")
    println(reflect.ArrayOf(10, intTyp).String())             // Output: [10]int
    println(reflect.ChanOf(reflect.BothDir, intTyp).String()) // Output: chan int
    println(reflect.MapOf(strTyp, intTyp).String())           // Output: map[string]int
    println(reflect.PtrTo(intTyp).String())                   // Output: *int
    println(reflect.SliceOf(intTyp).String())                 // Output: []int

    println(reflect.StructOf([]reflect.StructField{
        {Name: "IntA", Type: intTyp},
        {Name: "StrB", Type: strTyp},
    }).String()) // Output: struct { IntA int; StrB string }

    var err error
    errTyp := reflect.TypeOf(&err).Elem()
    println(reflect.FuncOf([]reflect.Type{intTyp}, nil, false).String())                            // Output: func(int)
    println(reflect.FuncOf(nil, []reflect.Type{intTyp}, false).String())                            // Output: func() int
    println(reflect.FuncOf([]reflect.Type{intTyp}, []reflect.Type{strTyp, errTyp}, false).String()) // Output: func() (string, error)
    println(reflect.FuncOf([]reflect.Type{reflect.SliceOf(strTyp)}, nil, true).String())            // Output: func(...string)
}
```

### 类型、值操作

- 所有的类型都可以用`Set`方法修改值。

类型 | 方法
---|---
实现接口 | `Type.Implements(Type) bool`
assignable | `Type.AssignableTo(Type) bool`
可转换类型 | `Type.ConvertibleTo(Type) bool`
可比较 | `Type.Comparable() bool`

#### basic types

类型 | 获取值 | 修改值
---|---|---
`int`, `int8`, `int16`, `int32`, `int64` | `Int() int64` | `SetInt(int64)`
`uint`, `uint8`, `uint16`, `uint32`, `uint64` | `Uint() uint64` | `SeUint(uint64)`
`string` | `String() string` | `SetString(string)`
`bool` | `Bool() bool` | `SetBool(bool)`
`float32`, `float64` | `Float() float64` | `SetFloat(float64)`
`[]byte` | `Bytes() []byte` | `SetBytes([]byte)`
`complex64`, `complex128` | `Complex() complex128` | `SetComplex(complex128)`

#### slice

操作 | 方法
---|---
创建 | `MakeSlice(Type, int, int) Value`
添加 | `Append(Value, ...Value) Value`、`AppendSlice(Value, Value) Value`
获取长度 | `Value.Len() int`
设置长度 | `Value.SetLen(int)`
获取容量 | `Value.Cap() int`
设置容量 | `Value.SetCap(int)`
切片 | `Value.Slice(int, int) Value`(也可用于`string`、`array`)、`Slice3`(也可用于`array`)
下标获取 | `Value.Index(int) Value`
拷贝 | `Copy(Value, Value)`
生成交换函数(给`sort`包用) | `Swapper(Value) func(int, int)`
是否为空 | `Value.IsNil() bool`

类型 | 方法
---|---
元素类型 | `Type.Elem() Type`

#### array

操作 | 方法
---|---
创建 | `New(Type) Value`
获取长度 | `Value.Len() int`

类型 | 方法
---|---
长度 | `Type.Len() int`
元素类型 | `Type.Elem() Type`

#### map

操作 | 方法
---|---
创建 | `MakeMap(Type) Value`、`MakeMapWithSize(Type) Value`
添加/删除 | `Value.SetMapIndex(key, val Value)`(删除时val为zero Value，可以写为`reflect.Value{}`)
所有的键 | `Value.MapKeys() []Value`
获取值 | `Value.MapIndex(Value) Value`
获取长度 | `Value.Len() int`
是否为空 | `Value.IsNil() bool`

类型 | 方法
---|---
键类型 | `Type.Key() Type`
元素类型 | `Type.Elem() Type`

#### chan

操作 | 方法
---|---
创建 | `MakeChan(Type, int) Value`
发送 | `Value.Send(Value)`、`Value.TrySend(Value) bool`
接收 | `Value.Recv() (Value, bool)`、`Value.TryRecv() (Value, bool)`
关闭 | `Value.Close()`
获取长度 | `Value.Len() int`
是否为空 | `Value.IsNil() bool`

类型 | 方法
---|---
方向 | `Type.ChanDir() ChanDir`
元素类型 | `Type.Elem() Type`

#### func

操作 | 方法
---|---
创建 | `MakeFunc(Type, func([]Value) []Value) Value`
调用 | `Value.Call([]Value)[]Value`、`Value.CallSlice([]Value) []Value`
是否为空 | `Value.IsNil() bool`

类型 | 方法
---|---
变长参数 | `Type.IsVariadic() bool`
参数个数 | `Type.NumIn() int`
参数类型 | `Type.In(int) Type`
返回值个数 | `Type.NumOut() int`
返回值类型 | `Type.Out(int) Type`


#### struct

操作 | 方法
---|---
创建 | `New(Type) Value`(`New`也可以用于创建基础类型如`int`等)
字段数量 | `Value.NumField() int`
获取字段 | `Value.Field(int) Value`、`Value.FieldByName(string) Value`、`Value.FieldByIndex([]int) Value`、`FieldByFunc(func(string) bool) Value`
方法数量 | `Value.NumMethod() int`
获取方法 | `Value.Method(int) Value`、`Value.MethodByName(string) Value`

类型 | 方法
---|---
方法数量 | `Type.NumMethod`
获取方法详情 | `Type.Method(int) Method`·`Type.MethodByName(string) Method`
是否实现接口 | `Type.Implements(Type) bool`
字段详情 | `Type.Field(int) StructField`、`Type.FieldByIndex([]int) StructField`、`Type.FieldByName(string) (StructField, bool)`、`Type.FieldByNameFunc(func(string) bool) (StructField, bool)`


#### ptr (interface类似)

操作 | 方法
---|---
创建 | `New(Type) Value`(`NewAt`不列，不推荐用`unsafe`包)
解指针 | `Value.Elem() Value`
是否为空 | `Value.IsNil() bool`

### 练习

#### 链表

试比较`makeList1`和`makeList2`：

```go
type node struct {
    Data interface{}
    Next *node
}

type list *node

func makeList1(ls *list, data ...interface{}) {
    p := ls
    for _, d := range data {
        *p = &node{d, nil}
        p = (*list)(&(*p).Next)
    }
}

func makeList2(ls interface{}, data ...interface{}) {
    p := reflect.ValueOf(ls).Elem()
    typ := p.Type().Elem()
    for _, d := range data {
        e := reflect.New(typ)
        e.Elem().FieldByName("Data").Set(reflect.ValueOf(d))
        p.Set(e)
        p = e.Elem().FieldByName("Next")
    }
}

func main() {
    var ls list
    // makeList1(&ls, 1, true, "Hello world")
    makeList2(&ls, 1, true, "Hello world")
    for p := ls; p != nil; p = p.Next {
        fmt.Println(p.Data)
    }
    // Output:
    // 1
    // true
    // Hello world
}
```

所以在Golang里面，反射编程只是将普通代码的流程写全了而已。

#### csv转json

将`a,b,c\n1,2,3\n4,5,6`转为json: `[{"a":"1","b":"2","c":"3"},{"a":"4","b":"5","c":"6"}]`

先看不用反射的写法：

```go
func csv2json(r *csv.Reader) []byte {
    var rows []map[string]string
    header, _ := r.Read()
    for {
        cols, err := r.Read()
        if err != nil {
            break
        }   
        m := make(map[string]string, len(header))
        for i := range header {
            m[header[i]] = cols[i]
        }   
        rows = append(rows, m)
    }
    p, _ := json.Marshal(rows)
    return p
}

func main() {
    var fileContent = "a,b,c\n1,2,3\n4,5,6"
    println(string(csv2json(csv.NewReader(strings.NewReader(fileContent)))))
}
```

用反射写法：

```go
func csv2json(r *csv.Reader) []byte {
    header, _ := r.Read()
    fields := make([]reflect.StructField, len(header))
    for i := range header {
        fields[i].Name = strings.Title(header[i])
        fields[i].Type = reflect.TypeOf("")
        fields[i].Tag = reflect.StructTag(`json:"` + header[i] + `"`)
    }
    typ := reflect.StructOf(fields)
    ls := reflect.New(reflect.SliceOf(typ)).Elem()
    for {
        cols, err := r.Read()
        if err != nil {
            break
        }
        val := reflect.New(typ).Elem()
        for i, col := range cols {
            val.Field(i).SetString(col)
        }
        ls = reflect.Append(ls, val)
    }
    p, _ := json.Marshal(ls.Interface())
    return p
}
```

这回在结构上和非反射写法不同了，好处是反射写法字段顺序是固定的，用`map[string]string`固然方便，但字段顺序有随机成分。


### 常见模式

用反射编程时，会有三类常见模式：
- 枚举：枚举不同类型，一般用`Value.Kind() Kind`；
- 循环：这里主要指，循环结构体字段；
- 递归：复用特定类型的特定处理流程。

一般流程会是：
- 入口函数**枚举**支持的类型，一般用`switch val.Kind() { case Ptr: ... }`；
- 每个`switch`的`case`下面对应一种类型的处理逻辑，`Array`、`Slice`、`Map`、`Struct`会**循环**每一个元素/字段；
- `Ptr`、`Interface`的`Elem`，和上面类型的每一个元素/字段一样，递归调用本身，直到最基本的`(U)Int`、`Float`、`String`、`Bool`等类型时，做具体逻辑；
- 特例：`time.Time`一般会特殊处理，比如会调用`time.Time.Format()`。

#### 例1：`DeepEqual`

具体代码参见：`$GOROOT/src/reflect/deepequal.go`，遍数一下，**枚举**、**递归**、**循环**，三者都用上了。

#### 例2：简易mysql orm

- 插入

一般插入语句结构为：`insert into table_abc (field_1, field_2, ...) values (value_1, value_2, ...)`，一般在Golang里面用orm，会将数据库表结构定义为一个`struct`，所以我们的设计很简单：`func SQLFieldAndValue(v interface{}) (fields []string, values []interface{})`，由调用者拼成sql执行。

```go
func SQLFieldAndValue(v interface{}) (fields []string, values []interface{}) {
    val := reflect.Indirect(reflect.ValueOf(v)) // 标记1
    if val.Kind() != reflect.Struct {
        panic("SQLFieldAndValue: only accept struct type variables")
    }

    typ := val.Type()
    for i := 0; i < val.NumField(); i++ {
        if !val.Field(i).CanInterface() {
            continue
        }
        fields = append(fields, typ.Field(i).Tag.Get("db"))
        values = append(values, val.Field(i).Interface())
    }
    return
}
```

标记1处，`reflect.Indirect(reflect.Value)`的意思是，如果外面传指针进来，忽略掉，直接解指针，因为我们对这个结构**只读**，所以不在意它是否是指针类型。

你也可以在里面加一些其他逻辑，比如像`json`一样，判断`omitempty`等。测试一下：

```go
type T struct {
    Id   int64  `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}

func main() {
    t := T{1, "eachain", 26}
    fields, values := SQLFieldAndValue(t)
    marks := make([]string, len(fields))
    for i := 0; i < len(marks); i++ {
        marks[i] = "?" 
    }
    sql := "insert into `table_abc`(`" +
        strings.Join(fields, "`, `") +
        "`) values(" +
        strings.Join(marks, ", ") + ")" 

    fmt.Println(sql)    // Output: insert into `table_abc`(`id`, `name`, `age`) values(?, ?, ?)
    fmt.Println(values) // Output: [1 eachain 26]
}
```

- 查询

查询语句结构一般为：`select (field_1, field_2, ...) from table_abc where ...`。获取字段名的过程和上面一样，另外我们需要注意的是，`database/sql`的接口是如何定义的：`sql.DB.Query(string, ...interface{}).Scan(fields ...interface{})`。这个接口要求我们得到对应结构体的每个字段。

```go
func SQLQueryFields(v interface{}) (dbFields []string, stFields []interface{}) {
    val := reflect.ValueOf(v)
    if val.Kind() != reflect.Ptr {
        panic("SQLQueryFields: the arg must be the variable address")
    }
    val = val.Elem()
    if val.Kind() != reflect.Struct {
        panic("SQLQueryFields: only accept struct type variables")
    }

    typ := val.Type()
    for i := 0; i < val.NumField(); i++ {
        if !val.Field(i).CanInterface() {
            continue
        }
        dbFields = append(dbFields, typ.Field(i).Tag.Get("db"))
        stFields = append(stFields, val.Field(i).Addr().Interface()) // 标记1
    }
    return
}
```

标记1处怎么理解？先看下面测试，用`fmt.Scan`代替测试：

```go
func main() {
    var t T
    dbFields, stFields := SQLQueryFields(&t)
    fmt.Println(dbFields) // Output: [id name age]
    // fmt.Sscanf("1 eachain 26", "%d %s %d", &t.Id, &t.Name, &t.Age) // 标记2
    fmt.Sscanf("1 eachain 26", "%d %s %d", stFields...)
    fmt.Println(t) // Output: {1 eachain 26}
}
```

对照着标记2处，再看标记1，`Scan`函数传的是地址，所以我们需要在标记1处取址返回，而不是值，这一点要注意。

---

### 写在最后

反射编程和普通编程在Golang中没有区别，反射只是普通代码的完全版，因此反射并没想象中的那么难。只要你知道Golang的语法规则，以及如何把代码写全了，反射对您来说，不是难事。

但，**请不要过度设计**。反射在Golang中是一大杀着，但学会用反射后，一定要克制自己，不要炫技。

在真实反射编程时，有人喜欢包容所有类型，不能说这样无法做到，只是这里劝诫：一定要对反射要操作的对象有限制，**限制支持的类型**。如果觉得这样做太死板，不能做到大而全，那就用Golang的接口**组合**做，而不是在反射上做。

比如，请实现一个`Copy(dst, src interface{})`函数，用反射实现将任意`src`拷贝到`dst`。如何实现？像下面这样写吗？

```go
func Copy(dst, src interface{}) {
    dstVal := reflect.ValueOf(dst)
    srcVal := reflect.ValueOf(src)
    switch src.Kind() {
    case reflect.Slice:
        switch dst.Kind() {
        case reflect.Slice:
            ...
        case reflect.Chan:
            ...
        case reflect.Map:
            ...
        case ...:
        }
    case ...:
    }
}
```

谬矣！做到大而全是不太可能的，而且由于这样做是类型数量的乘方。如何测试还是个问题。这种时候不应该用反射，而是用接口：

```go
type Iterator interface {
    Range(func(interface{}))
}

type Writer interface {
    Store(interface{})
}

func Copy(dst Writer, src Iterator) {
    src.Range(dst.Store)
}
```

简单清晰，根本用不着反射。而在生成`Iterator`和`Writer`的时候，可以用反射：

```go
type sliceIterator struct {
    reflect.Value
}

func (ls sliceIterator) Range(f func(interface{})) {
    n := ls.Len()
    for i := 0; i < n; i++ {
        f(ls.Index(i).Interface())
    }
}

func SliceIterator(slice interface{}) Iterator {
    ls := reflect.Indirect(reflect.ValueOf(slice))
    switch ls.Kind() {
    case reflect.Array, reflect.Slice:
    default:
        panic("SliceIterator: only accept 'slice' and 'array' type")
    }
    return sliceIterator{ls}
}
```

```go
type sliceWriter struct {
    reflect.Value
}

func (ls *sliceWriter) Store(v interface{}) {
    ls.Set(reflect.Append(ls.Value, reflect.ValueOf(v)))
}

func SliceWriter(slice interface{}) Writer {
    ptr := reflect.ValueOf(slice)
    if ptr.Kind() != reflect.Ptr {
        panic("SliceWriter: the arg should be a pointer")
    }
    ls := ptr.Elem()
    switch ls.Kind() {
    case reflect.Slice:
    default:
        panic("SliceWriter: only accept 'slice' type")
    }
    return &sliceWriter{ls}
}
```

```go
func main() {
    src := []int{1, 2, 3}
    var dst []int
    Copy(SliceWriter(&dst), SliceIterator(src))
    fmt.Println(dst) // Output: [1 2 3]
}
```


这样做的意义在于任何人可以按实际需要扩展`Copy`函数并实现特殊功能，而不局限于像上面枚举类型时的所有情况。

那什么情况下适合用反射呢：**逻辑一致**
- 不论什么类型，逻辑一致，例如`json.Marshal`；
- 某一特定类型，逻辑一致，例如数据库结构对应的代码`struct`。

不适合用反射的情况：**逻辑各异**
- 每个元素/字段逻辑各异；
- 穷举所有可能性太复杂。

像上面`Copy`函数，虽然是“无论什么类型，逻辑一致”，但同样，想要穷举所有可能性，几乎不可能，而且说不好就会出现逻辑冲突，他人理解起来也是难题。

所以，**请不要过度设计**。反射在Golang中是一大杀着，但学会用反射后，一定要克制自己，不要炫技。

