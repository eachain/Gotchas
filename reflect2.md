# Go reflect 反射实例解析

*——教坏小朋友系列*


## 0 FBI WARNING

**对于本文内容，看懂即可，完全不要求学会。**

---

## 1 现象分析

网上关于golang反射的文章多如牛毛，可实际用反射的人，寥寥可数。

我分析大家不用反射的原因，大概有两点：

- 不会
- 笨重

### 1.1 不会
网上关于反射的文章很多，可多数都是在分析原理，并没有给出实例。而反射恰好处在一个很尴尬的点上：懂了原理不等于会用。

于是乎，大多数人高喊口号：“官方不推荐用这个库！”是的，官方不推荐用的库有两个：`reflect`和`unsafe`（严格意义上来说，还有`cgo`）。

附[官方说法](https://blog.golang.org/laws-of-reflection)：

```
It's a powerful tool that should be used with care and avoided unless strictly necessary.
```

可是我又发现一个奇怪现象，看看下面这段代码：

```go
func bytes2str(p []byte) string {
	return *(*string)(unsafe.Pointer(&p))
}
```
我不知道这段代码是从哪里流传出去的（疑似官方库`strings.Builder`），然后这段代码就被玩烂了，说好的不推荐使用`unsafe`呢？

**善意的提醒：**
上面这段代码和`bufio.Scanner`或`bufio.Reader`共用的时候，将出现彩蛋。看下面示例：

```go
func main() {
    s := "aaa\nbbb\ncc"
    sc := bufio.NewScanner(strings.NewReader(s))
    sc.Buffer(make([]byte, 4), 4)

    sc.Scan()
    t := bytes2str(sc.Bytes())
    fmt.Println(t) // Output: aaa

    sc.Scan()
    fmt.Println(t) // Output: bbb

    sc.Scan()                                                                                 
    fmt.Println(t) // Output: ccb
}
```
和您想象中的结果一样吗？如果一样，恭喜您。

书接前文，言归正传。最后，我想，大家嘴里说着不用不用，其实不是不想用，而是***不会用***。当大家真正拿到实例，知道怎么用的时候，“真香定律”便出现了。

反射也是一样，大家都说反射不好理解、性能差、容易出错，其实还不是因为不会用？所以呢，今天我不讲太多原理了，直接上干货，给出一些实例或模板出来，后面再遇到类似情况，就像上面`bytes2str`一样，拿来套用即可。

**又是一段善意的提醒：**
建议大家要学好`闭包`和`接口`，这样在用反射的时候，将会更加得心应手（如果您能掌握`unsafe`，那真真儿是极好的）。

### 1.2 笨重

大家在使用反射时，都在默认一个“无知”概念：我什么都不知道，所以我要枚举所有可能，我要实现所有可能。第一个“所有可能”要求我们检查所有可能的输入参数是否合法，第二个“所有可能”要求我们实现所有可能参数的处理逻辑。

为什么要仔细检查所有可能输入的参数？因为`reflect`库稍加不慎就会panic。

为什么要实现所有可能参数的处理逻辑呢？参照官方库`json`、`reflect.DeepEqual`的实现（其反射代码之多、之复杂，让人望而却步），反射不就应该这样用吗？

但仔细想想，我们真的需要这两个“所有可能”吗？不一定。如果我们想做到“大而全”，那确实需要两个“所有可能”，但如果我们不要“大而全”呢？我们拿反射处理特定情况行不行呢？

不得不说，正是这个无知概念，让大家在使用反射时，像是戴着镣铐在针尖上跳舞，如履薄冰，战战兢兢。今天，我们暂且抛却这个概念，化巨阙为瑞士军刀。

**从现在开始，我们准确的知道传进来的参数是什么。如果不知道，很简单，限制死。**

---


## 2 由高斯求和说起

请解决以下问题：写程序输出1+2+3+...+10的结果。

```go
func main() {
    var sum int
    for i := 1; i <= 10; i++ {
        sum += i
    }
    println(sum)
}
```

如果把这段代码抽成函数呢？

```go
func Gauss(n int) int {
    var sum int
    for i := 1; i <= n; i++ {
        sum += i
    }
    return sum
}
```

请问，`Gauss`函数为什么需要一个参数`n`？很简单，这样该函数不止可以求从1加到10的结果，也可以求从1加到100，加到1000，10000的结果，而不用每次求和再重新写段代码。也就是说，该函数在某种情况下是通用的：求从1加到n的和。

通用性，使得同一段代码可以在不同的地方被反复调用，而不用每次都重新写一遍相同的逻辑。

仔细观察，你会发现一个秘密：99%的通用性代码，是对“值”的通用。值是什么？比如`var a int = 123`，变量a的值是123。

还记得有种数据结构叫哈希吗？用哈希的时候，要抛弃传统`通过下标找值`的思维，转向`通过值找下标`的思维，这样哈希才能玩的贼6。

反射也是一样，写反射代码时不要总想着对`变量的值`操作，这时候需要同时对`变量的值和类型`进行操作。比如像上面的`Gauss`函数，可以对不同的n值进行求和，n的限制是大于等于0。对变量值的操作，大家都很熟悉。学习反射，最主要的，是学会对变量类型的操作。本文中所有反射例子，都会对类型进行限制，这也是用好反射的一个关键。

在接下的例子中，您可以试着从这个角度出发，去理解代码的意图。

---

## 3 实例

以下实例，或模板，都是针对官方库的使用。大家可以重点关注一下对类型的操作。

### 3.1 http转rpc模板

大家用go语言写的最多的，应该就是web应用了，写web应用的时候，大家又陷入一个怪圈中：有没有一个好的框架？大家找来找去，发现还是gin和echo好用。

这里，为大家提供一种基于反射的http模板，窃以为比gin和echo还要人性化一点。

**技能点：**
- 闭包
- 装饰器
- 反射

#### 3.1.1 闭包

go语言中闭包有两种形式：
- 实例的方法
- 函数嵌套

##### 3.1.1.1 实例的方法

```go
type T string

func (t T) greating() {
    println(t)
}

t := T("hello world!")
g := t.greating
g()
```

可以看到，变量`g`和`t`已经没有关系，但`g`还是可以访问`t`的内容，这是比较简单的一类闭包实现，但不常用，因为它总是可以被第二种形式替代。

##### 3.1.1.2 函数嵌套

还是用一段广为流传的代码作示例好了：

```go
func Incr() func() int {
    var i int
    return func() int {
        i++
        return i
    }
}

incr := Incr()
println(incr()) // Output: 1
println(incr()) // Output: 2
println(incr()) // Output: 3
```

由此可见，闭包本身不难理解，是不是就像`1+1=2`一样简单？好了，下面我们将用它推导微积分（手动狗头）。


#### 3.1.2 装饰器

在go中，几乎所有的接口，都可以使用装饰器。比如常用的`io.LimtedReader`、`context.Context`等。http库，要处理http请求，就要实现`http.Handler`接口，实现该接口的方式有很多，http库给出了非常方便一种：`http.HandlerFunc`，接下来，我们用它来实现http装饰器。

http库对于`HandlerFunc`的定义如下：

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)
```

这种形式的定义，注定我们将要用闭包的第二种形式：

```go
func WithRecovery(handler http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if e := recover(); e != nil {
                log.Printf("uri = %v, panic error: %v, stack: %s", r.URL.Path, e, debug.Stack())
                w.WriteHeader(http.StatusInternalServerError)
                w.Write([]byte(http.StatusText(http.StatusInternalServerError)))
            }   
        }()                                                                                   
        handler.ServeHTTP(w, r)
    })  
}
```

使用方法：

```go
http.NewServeMux().Handle("/ping", WithRecovery(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("pong"))
})))
```

如果您看过`gin`或`echo`的代码，就会明白，装饰器（在http中或叫middleware）的实现方式大同小异，比如`gin.New().Use()`。这些第三方库最大的好处在于，写法上更简捷易懂。

#### 3.1.3 反射

不知道大家发现没有，大多数的第三方库，包括gin、echo，他们都是在解决路由和中间件的问题。但大家在处理逻辑时，拿到的还是最原始的`http.Request`和`http.ResponseWriter`，如果我们看过官方的`rpc`库，就会想，能不能把http请求转成rpc格式呢？答案是：当然可以，用反射！我们先定义用法，再写胶水代码。假设用法如下：

```go
type Context struct{}  // 一些http信息
type PingReq struct{}  // 参数
type PingResp struct{} // 业务返回值
// 最终业务逻辑函数格式：
func Ping(ctx *Context, req *PingReq, resp *PingResp) Error {
    return nil
}
```

参照上述格式，我们来完成胶水逻辑：
- 获取函数类型，从而获取函数入参类型;
- 生成`*Context`、`*PingReq`、`*PingResp`;
- 将函数包装成`http.Handler`;
- 调用函数，接收返回值;
- 输出结果。

请先在心里默念：“***`我知道参数是什么`***”。OK，看主要代码（后面有完整版）：

```go
// function should be like:
//     func Ping(*Context, *PingReq, *PingResp) Error { return nil }
func WithFunc(function interface{}) http.Handler {
    fn := reflect.ValueOf(function) // function代表Ping
    fnTyp := fn.Type() // 获取函数类型，从类型出发，创建变量
    // Context是http信息，所有业务逻辑共用的数据类型，可以不用反射
    // 这里获取Type时解指针，下面reflect.New的时候拿到的是指针值
    arg1 := fnTyp.In(1).Elem() // type: PingReq
    arg2 := fnTyp.In(2).Elem() // type: PingResp

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := &Context{} // 填入http请求信息
        req := reflect.New(arg1) // 创建请求变量，type: *PingReq
        json.NewDecoder(r.Body).Decode(req.Interface()) // 这步看项目需要自行处理
        resp := reflect.New(arg2) // 创建返回值变量，type: *PingResp
        // 函数调用，传入预定的三个参数，接收Error返回值
        out := fn.Call([]reflect.Value{reflect.ValueOf(ctx), req, resp})
        if ret := out[0].Interface(); ret != nil {
            err := ret.(Error)
            Render(w, err.Code(), err.Error(), nil)
            return
        }
        Render(w, 0, "", resp.Interface())
    })
}
```

完整代码：

```go
type Context struct {
    Trace  string
    API    string
    Func   string
    Header http.Header
    Query  url.Values
    Uid    string
}

type Error interface {
    error
    Code() int 
}

type Middleware func(*Context) Error

// 不相关逻辑从简
func Render(w http.ResponseWriter, code int, msg string, data interface{}) {
    json.NewEncoder(w).Encode(map[string]interface{}{
        "errcode": code,
        "errmsg": msg,
        "data": data,
    })
}

// function should be like:
//     func Ping(*Context, *PingReq, *PingResp) Error { return nil }
func WithFunc(function interface{}, mws ...Middleware) http.Handler {
    fn := reflect.ValueOf(function)
    fnName := runtime.FuncForPC(fn.Pointer()).Name()
    fnTyp := fn.Type()
    arg1 := fnTyp.In(1).Elem()
    arg2 := fnTyp.In(2).Elem()

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := &Context{
            Trace:  r.URL.Query().Get("trace_id"),
            API:    r.URL.Path,
            Func:   fnName,
            Header: r.Header,
            Query:  r.URL.Query(),
        }
        for _, mw := range mws {
            err := mw(ctx)
            if err != nil {
                mwName := runtime.FuncForPC(reflect.ValueOf(mw).Pointer()).Name()
                log.Printf("WithFunc middleware %v ctx %+v: %v", mwName, ctx, err)
                Render(w, err.Code(), err.Error(), nil)
                return
            }
        }

        req := reflect.New(arg1)
        json.NewDecoder(r.Body).Decode(req.Interface())
        resp := reflect.New(arg2)
        out := fn.Call([]reflect.Value{reflect.ValueOf(ctx), req, resp})
        if ret := out[0].Interface(); ret != nil {
            err := ret.(Error)
            Render(w, err.Code(), err.Error(), nil)
            return
        }
        Render(w, 0, "", resp.Interface())
    })
}
```

测试：

```go
type PingReq struct{}
type PingResp struct{}
func Ping(ctx *Context, req *PingReq, resp *PingResp) Error {
    return nil
}

http.NewServeMux().Handle("/ping", WithFunc(Ping))
```

这样写起逻辑来，是不是感觉清爽多了，而且不用关心底层是不是http了？


### 3.2 简易ORM

在一般项目中，用到数据库，比如mysql，是非常常见的事。大家用数据库想到的第一件事，就是找个趁手的orm。网上比较流行的是`gorm`和`xorm`等。

那如果不用gorm和xorm，自己能不能写个简单实用的orm呢？

**技能点：**
- 反射
- 封装

#### 3.2.1 反射

官方库`database/sql`提供的接口很原始，比如读数据，就像`fmt.Scanf`一样用，但项目代码中，用`scanf`的方式，会不会有点太原始了？比如像下面一样：

```go
type T struct {
    Id   int64  `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}

func main() {
    db, _ := sql.Open("mysql", "xxx")
    row, _ := db.QueryRow("select (id, name, age) from tbl where id=123")
    var t T
    row.Scan(&t.Id, &t.Name, &t.Age)
    fmt.Println(t)
}
```

这样写一个两个表还OK，多了之后肯定要崩溃，对每张表都要写一套如何读取数据的代码，明明逻辑都是一样的。这种枯燥的工作就很容易出错。另外，加个字段要考虑数据库字段和程序字段的一一严格对应。

我想，在多数人心目中，这样原始的接口不太好用。所以很多人选择了orm。那如果没有orm，我们能不能把原始接口变的好用起来呢？答案是：当然可以。我们把这套相同的逻辑抽出来，用反射忽略类型hard code，即可适应所有数据的读取工作。先看相同的逻辑：

```go
var t T
fields := []string{"id", "name", "age"}
values := []interface{}{&t.Id, &t.Name, &t.Age}
sql := fmt.Sprintf("select (`%v`) from tbl where id=123", strings.Join(fields, "`, `"))
row, _ := db.QueryRow(sql)
row.Scan(values...)
```

由上面逻辑，我们可以看出，我们需要的是`fields`和`values`，我们只需要通过反射拿到这两个数组即可。由此胶水逻辑为：
- 获取参数类型;
- 遍历参数`struct`每个字段;
- 通过每个字段得到数据库字段名和该字段地址;
- 返回数据库字段名列表和字段地址列表。

后续逻辑会通过字段名和字段地址读取数据库数据：

```go
func SQLQueryFields(v interface{}) (fields []string, values []interface{}) {
    val := reflect.Indirect(reflect.ValueOf(v))
    typ := val.Type()
    for i := 0; i < val.NumField(); i++ {
        if !val.Field(i).CanInterface() {
            continue
        }
        // 数据库字段名
        fields = append(fields, typ.Field(i).Tag.Get("db"))
        // 注意是指针，需要取址操作
        values = append(values, val.Field(i).Addr().Interface())
    }
    return
}
```

可见，代码量非常之少。谁说反射很笨重的？来看使用方法：

```go
func main() {
    var t T
    fields, values := SQLQueryFields(&t)
    fmt.Println(fields) // Output: [id name age]
    // fmt.Sscanf("1 eachain 26", "%d %s %d", &t.Id, &t.Name, &t.Age)
    fmt.Sscanf("1 eachain 26", "%d %s %d", values...)
    fmt.Println(t) // Output: {1 eachain 26}
}
```

接下来就是如何读多条记录的问题了：
- 通过参数（列表）逐级获取真实元素类型；
- 生成一个新元素，按上面逻辑读取；
- 将新元素追加到列表中。

```go
// slice type: *[]T or *[]*T
func QueryRows(rows *sql.Rows, slice interface{}) error {
    ls := reflect.ValueOf(slice).Elem() // type: []T or []*T
    elemTyp := ls.Type().Elem() // type: T or *T
    isPtr := false
    if elemTyp.Kind() == reflect.Ptr {
        elemTyp = elemTyp.Elem() // type: T
        isPtr = true
    }

    for rows.Next() {
        elem := reflect.New(elemTyp) // type: *T
        _, fields := SQLQueryFields(elem.Interface())
        err := rows.Scan(fields...)
        if err != nil {
            return err 
        }
        // 以下代码翻译成正常代码即： ls = append(ls, elem)
        if isPtr {
            ls.Set(reflect.Append(ls, elem))
        } else {
            ls.Set(reflect.Append(ls, elem.Elem()))
        }
    }

    return rows.Err()
}
```

是不是觉得写个orm没有想象中的辣么难？

#### 3.2.2 封装

封装不是本文重点，这里不作具体介绍。但有了上面基础，封装一个类似`gorm`这样`xxx.Select(fields).From(table).Where(cond).OrderBy(order).Limit(limit).Rows(&slice)`的链式调用，应该不难。

同样的`insert`、`update`、`delete`等操作是一样的，这里不再重复。

**注意，本节到此为止。**
这只是个orm的雏形，点到即可，要想得到一个完善的orm，最终还是要写成类似标准库`encoding/json`或三方库`gorm`那种形式的。但那已经不是本文要关心的点了。


### 3.3 兼容不同格式的API返回结果

在对接一些系统的时候，会发现接口返回格式各异，例如：

```
{"errcode": 1, "errmsg": "some error", "data": {"a": 789}}
{"err_code": 1, "err_msg": "some error", "data": {"a": 789}}
{"errcode": "1", "err_msg": "some error", "a": 789}
```

相信很多人遇到这种情况时，想到的第一件事不是怎么解决，而是WTF。这种结果，能不能兼容呢？

**提醒：**
这种情况用以下代码是不可行的：

```go
var errcode json.Number
var errmsg string
m := map[string]interface{}{
    "errcode":  &errcode,
    "err_code": &errcode,
    "errmsg":   &errmsg,
    "err_msg":  &errmsg,
}
json.Unmarshal(p, &m)
```

**技能点：**
- 接口
- 反射

#### 3.3.1 接口

*由于这里主要在说json数据处理，所以在这只提`json.Unmarshaler`接口了。*

在`encoding/json`中，`Unmarshaler`接口允许外部可以按自定义的格式解析json数据，这给了我们兼容不同json格式的可能。比如`json.Number`。同样的，我们也可以写一个`ErrCode`以兼容数字、字符串格式的json返回结果：

```go
type ErrCode int 

func (ec *ErrCode) UnmarshalJSON(p []byte) (err error) {
    if p[0] == '"' {
        p = p[1 : len(p)-1]
    }   
    *(*int)(ec), err = strconv.Atoi(string(p))
    return
}
```
如果发现p是字符串，我们去掉双引号后按数字解析即可。

#### 3.3.2 反射

我们首先明确一下要解决的问题：
- 兼容字段名errcode和err_code两种格式，同理errmsg；
- `data`有时在json的data字段中，有时在外面平铺展开。

我们在一开始就提到了，用`map`是不能解决这个问题的，那如果用`struct`呢？我们需要做什么：
- 需要两个字段，一个是errcode，一个是err_code，但由指针指向相同的地址。这样无论出现哪个，最终都将解析到同一个值里面；
- 将`data`放在json的data字段，同时将`data`展开到外层。这样无论哪种情况，都会被解析到，另一种被忽略；
- 生成一个全新的struct，将指针指向传入参数的内存地址，并返回。

来看示例代码：

```go
type CallError struct {
    ErrCode int
    ErrMsg  string
}

func (ce CallError) Error() string {
    return strconv.Itoa(ce.ErrCode) + ": " + ce.ErrMsg
}

func Join(err *CallError, data interface{}) interface{} {
    var fields []reflect.StructField
    var values []reflect.Value

    // 第一个errcode，类型是*ErrCode，兼容数字和字符串，并指向err.ErrCode
    fields = append(fields, reflect.StructField{
        Name: "ErrCode1",
        Tag:  `json:"errcode"`,
        Type: reflect.TypeOf(new(ErrCode)), // 类型是指针
    })
    values = append(values, reflect.ValueOf((*ErrCode)(&err.ErrCode)))

    // 第二个err_code，类型同样是*ErrCode，也指向err.ErrCode
    fields = append(fields, reflect.StructField{
        Name: "ErrCode2",
        Tag:  `json:"err_code"`,
        Type: reflect.TypeOf(new(ErrCode)),
    })
    values = append(values, reflect.ValueOf((*ErrCode)(&err.ErrCode)))

    // 第一个errmsg，指向err.ErrMsg
    fields = append(fields, reflect.StructField{
        Name: "ErrMsg1",
        Tag:  `json:"errmsg"`,
        Type: reflect.TypeOf(new(string)),
    })
    values = append(values, reflect.ValueOf(&err.ErrMsg))

    // 第二个err_msg，也指向err.ErrMsg
    fields = append(fields, reflect.StructField{
        Name: "ErrMsg2",
        Tag:  `json:"err_msg"`,
        Type: reflect.TypeOf(new(string)),
    })
    values = append(values, reflect.ValueOf(&err.ErrMsg))

    // data字段，并将参数data置于此
    fields = append(fields, reflect.StructField{
        Name: "Data",
        Tag:  `json:"data"`,
        Type: reflect.TypeOf(new(interface{})).Elem(), // 类型是interface{}
    })
    values = append(values, reflect.ValueOf(data))

    // 将参数data所有字段展开放到最外层
    v := reflect.Indirect(reflect.ValueOf(data))
    if v.Kind() == reflect.Struct { // 标注
        t := v.Type()
        for i := 0; i < t.NumField(); i++ {
            f := t.Field(i)
            f.Type = reflect.PtrTo(f.Type) // 类型是指针
            fields = append(fields, f)
            values = append(values, v.Field(i).Addr())
        }
    }

    // 生成新struct，并将指针指向参数
    dst := reflect.New(reflect.StructOf(fields))
    v = dst.Elem()
    for i := 0; i < len(values); i++ {
        v.Field(i).Set(values[i])
    }
    // 返回可被json.Umarshal的对象
    return dst.Interface()
}
```

上面*标注*的地方是通用写法，还有一种不通用的简便写法：

```go
t := v.Type()
fields = append(fields, reflect.StructField{
    Anonymous: true,
    Name:      t.Name(),
    Type:      reflect.PtrTo(t),
})
values = append(values, v.Addr())
```

这种写法要求`data`不能带有`Method`，如果有，将panic。

有一点我没在注释里面提及：**所有字段均是以指针形式出现**。为什么要这样做？

设想：如果新struct各字段不用指针，当我们`reflect.New(reflect.StructOf(fields))`的时候，go会为该struct分配空间。当我们解析json的时候，最终结果都将解析到新生成的struct里面去，也就是说，值会被写入新分配的内存空间，而不是我们传入参数的内存空间。竹篮打水一场空，这不是我们想要的结果。

可如果我们用指针，我们可以任意指定指针指向的空间。并且指针符合[反射第三定律](https://blog.golang.org/laws-of-reflection#TOC_8.)“可写”的条件。最终我们将新生成struct的各个字段指向我们希望的内存空间：传入的参数。这样，当json解析的时候，会将结果写入到我们想要的内存空间中。

### 3.4 公共字段操作

***前排警告：***
**本节内容超纲，不要求看懂。**

请求的返回结果中，有一些相同字段，需要在接口返回的时候，自动填充这些字段的值，比如：

```go
type Resp struct {
    Time   int64  `json:"time"`
    Server string `json:"server"`
    Num    int    `json:"num"`
}

type Resp2 struct {
    Time   int64  `json:"time"`
    Server string `json:"server"`
    Name   string `json:"name"`
}

type Resp3 struct {
    Time   int64  `json:"time"`
    Server string `json:"server"`
    WTF    string `json:"wtf"`
}
```

要求：`Time`自动填充当前时间，`Server`自动填充`hostname`。
真实情况是：返回结果，有可能是指针`*Resp`，有可能不是指针`Resp`。

如果是指针，用反射很好解决，因为字段都是`CanSet()`的，但如果出现不是指针的情况呢？为什么现实总是如此残酷？也罢，会反射的我们总是可以见招拆招。

大家想到的第一个方案，很可能是深度拷贝(`DeepCopy`)，这段代码在github上有人实现，通过深度拷贝，获取一个可写的`reflect.Value`，从而写入`Time`和`Server`。提前声明，DeepCopy是正规解决方案。但本文一开始说了，不会用这种巨无霸的实现方式，我们要另辟蹊径。

附DeepCopy方案示例代码：

```go
func autoSetTimeAndServer(resp interface{}) {
    val := reflect.ValueOf(resp)
    if val.Kind() != reflect.Ptr {
        val = reflect.New(val.Type())
        DeepCopy(val.Interface(), resp)
    }
    val = val.Elem()

    val.FieldByName("Time").SetInt(time.Now().Unix())
    hostname, _ := os.Hostname()
    val.FieldByName("Server").SetString(hostname)
}
```

**技能点：**
- unsafe
- 反射

#### 3.4.1 unsafe

本文一开始就提到了unsafe，即上文的`bytes2str`，要用unsafe，需要对go数据底层存储有一定的认知。比如为什么可以直接把`[]byte`通过unsafe转成`string`，反过来由`string`转`[]byte`行不行？

**`uintptr`和`unsafe.Pointer`有什么区别？**

`uintptr`是一个变量，`unsafe.Pointer`是一个指针。这意味着如果GC在移动内存的时候，会更新`unsafe.Pointer`，因为它是指针。而不会修改`uinptr`，因为它只是个普通变量。因此，**不要用`uintptr`的变量保存内存地址，但可以用它保存偏移量**。内存地址可能会因为GC而变化，偏移量不会。

我们将要解决的这个问题，最终传入的参数，肯定是以`interface{}`的形式传入，该`interface{}`里面可能是`Resp`、`*Resp`、`Resp2`、`*Resp2`等等各种情况，那我们先看一下`reflect`库中关于`interface{}`的定义：

```go
type emptyInterface struct {
    typ  *rtype
    word unsafe.Pointer
}
```

一个`interface{}`由两部分组成：`typ`、`val`(`word`)。注意看，`word`是个指针！这意味着什么？`CanSet`，是的，从某种意义上来说，它是可以被修改的。我们做个实验：

```go
type iface struct {
    typ unsafe.Pointer // 我们知道类型信息，忽略该值
    val unsafe.Pointer // 这里我将字段名换成了val
}

func echo(x interface{}) {
    println(x.(int)) // Output: 123
    *(*int)((*iface)(unsafe.Pointer(&x)).val) = 456 // 标注
    println(x.(int)) // Output: 456
}

func main() {
    var i int = 123
    echo(i)
    println(i) // 想想输出多少，为什么？ // Output: 123
}
```

我们知道，如果将`标注`的代码换成`reflect.ValueOf(x).SetInt(456)`，程序将panic，因为变量`x`不是`CanSet`的。所以我们可以看到，`reflect`不能做到事（指`x.SetInt`），`unsafe`做到了。

#### 3.4.2 反射

有了上面的基础，相信这一步非常简单了，我们可以得到`*Resp`的地址，要修改其中某个字段的值，还需要一样东西：偏移量。我们将通过反射得到它：

```go
func autoSetTimeAndServer(resp interface{}) {
    typ := reflect.TypeOf(resp)
    if typ.Kind() == reflect.Ptr {
        typ = typ.Elem()
    }
    timeField, _ := typ.FieldByName("Time")
    serverField, _ := typ.FieldByName("Server")
    i := (*iface)(unsafe.Pointer(&resp))
    *(*int64)(unsafe.Pointer(uintptr(i.val) + timeField.Offset)) = time.Now().Unix()
    *(*string)(unsafe.Pointer(uintptr(i.val) + serverField.Offset)), _ = os.Hostname()
}

func send(resp interface{}) {
    autoSetTimeAndServer(resp)
    p, _ := json.Marshal(resp)
    fmt.Println(string(p))
}

func main() {
    send(&Resp{Num: 123})
    // Output: {"time":1587802664,"server":"WTF","num":123}

    send(Resp{Num: 456})
    // Output: {"time":1587802664,"server":"WTF","num":456}
}
```

本例用到的反射知识不多，大多数是需要掌握`unsafe`才能完成的操作，但将两者结合起来，会有一定难度。所以这里我将项目中实际遇到的情况精简了很多，这样大家可以更方便理解。

#### 3.4.3 类型

前面小节中介绍的方法是非常容易理解的一种方法，不知道大家有没有发现：该方法是在忽略类型的情况下实现需求的。我们在第二部分说了，反射最重要的是学会对类型的操作。下面我们将介绍对类型操作以实现需求。

在说具体操作前，我们需要先看`reflect`库的源码：

```go
func TypeOf(i interface{}) Type {
    eface := *(*emptyInterface)(unsafe.Pointer(&i))
    return toType(eface.typ)
}

func toType(t *rtype) Type {
    if t == nil {
        return nil
    }
    return t
}
```

简化一下：

```go
func TypeOf(i interface{}) Type {
    return (*emptyInterface)(unsafe.Pointer(&i)).typ
}
```

由此可见，一个`interface{}`中天然包含反射信息！下面我们自己组装一个`reflect.Type`试试（**注意`reflect.Type`本身是`interface{}`**）：

```go
func autoSetTimeAndServer(resp interface{}) {
    typ := reflect.TypeOf("")
    t := (*iface)(unsafe.Pointer(&typ)) // 获取*reflect.rtype类型
    r := (*iface)(unsafe.Pointer(&resp))
    // 生成一个新的interface{}:
    //   typ是*reflect.rtype,确保该interface{}实现了reflect.Type;
    //   val是resp的类型，实际上我们是对该类型进行操作.
    // 这个问题倒着想会比较轻松:
    //   将一个*reflect.rtype类型的值转成interface{}
    //   iface的typ和val各是什么?
    typ = *(*reflect.Type)(unsafe.Pointer(&iface{t.typ, r.typ}))
    if typ.Kind() == reflect.Ptr {
        typ = typ.Elem()
    }

    timeField, _ := typ.FieldByName("Time")
    serverField, _ := typ.FieldByName("Server")
    *(*int64)(unsafe.Pointer(uintptr(r.val) + timeField.Offset)) = time.Now().Unix()
    *(*string)(unsafe.Pointer(uintptr(r.val) + serverField.Offset)), _ = os.Hostname()
}
```

如果您已经理解了上述操作，马上您就会意识到，没必要那么麻烦，我们直接修改`resp`的类型不就行了：

```go
func autoSetTimeAndServer(resp interface{}) {
    typ := reflect.TypeOf(resp)
    if typ.Kind() != reflect.Ptr {
        // 将类型变为指针，以使resp CanSet
        typ = reflect.PtrTo(typ)
        (*iface)(unsafe.Pointer(&resp)).typ =
            (*iface)(unsafe.Pointer(&typ)).val
    }
    // 现在resp肯定是指针，即val.CanSet()肯定为true
    val := reflect.ValueOf(resp).Elem()

    val.FieldByName("Time").SetInt(time.Now().Unix())
    hostname, _ := os.Hostname()
    val.FieldByName("Server").SetString(hostname)
}
```

既然已经到这种程度了，那直接修改`reflect.Value`可不可行呢？当然也可以，这里不再赘述，只给出示例：

```go
type rvalue struct {
    typ  unsafe.Pointer
    ptr  unsafe.Pointer
    flag uintptr
}

func autoSetTimeAndServer(resp interface{}) {
    val := reflect.ValueOf(resp)
    if val.Kind() != reflect.Ptr {
        typ := reflect.PtrTo(val.Type())
        rv := (*rvalue)(unsafe.Pointer(&val))
        rv.typ = (*iface)(unsafe.Pointer(&typ)).val
        rv.flag = uintptr(reflect.Ptr)
    }
    val = val.Elem()

    val.FieldByName("Time").SetInt(time.Now().Unix())
    hostname, _ := os.Hostname()
    val.FieldByName("Server").SetString(hostname)
}
```

---

## 4 反射练习

关于反射的理论文章很多，大家在看完理论后，会产生一种“我觉得我又行了”的错觉。当实际用到反射的时候，却发现，理论，离真刀真枪的实战还有一定距离。所以，我在这里给出一种练习方式：把普通代码翻译成反射代码。

### 4.1 chan操作

**本例旨在：**
体现普通代码和反射代码别无二致。（**反例例外**）

我们经常会用chan当缓冲队列用，举个最简单的例子：

```go
ch := make(chan int)

go func() {
    for i := 0; i < 3; i++ {
        ch <- i
    }
    close(ch)
}() 

for n := range ch {
    println(n)
}
```

将上面代码翻译成反射代码，该怎么写呢？很简单，不需要解释：

```go
ch := reflect.MakeChan(reflect.ChanOf(reflect.BothDir,
    reflect.TypeOf(int(0))), 0)

go func() {
    for i := 0; i < 3; i++ {
        ch.Send(reflect.ValueOf(i))
    }
    ch.Close()
}()

for {
    n, ok := ch.Recv()
    if !ok {
        break
    }
    println(n.Int())
}
```

下面给出一个*反例*，用正常代码不好实现，用反射却异常轻松的一段代码(非重点，不解释)：

```go
func send(chs interface{}, value interface{}) int {
    val := reflect.ValueOf(value)
    ls := reflect.Indirect(reflect.ValueOf(chs))
    n := ls.Len()
    sc := make([]reflect.SelectCase, 0, n)
    for i := 0; i < n; i++ {
        sc = append(sc, reflect.SelectCase{
            Chan: ls.Index(i),
            Dir:  reflect.SelectSend,
            Send: val,
        })  
    }   
    sc = append(sc, reflect.SelectCase{
        Dir:  reflect.SelectDefault,
    })  
    i, _, _ := reflect.Select(sc)
    if i == len(sc)-1 {
        return -1 // default case
    }
    return i
}

func recv(chs interface{}, dst interface{}) int {
    ls := reflect.Indirect(reflect.ValueOf(chs))
    n := ls.Len()
    sc := make([]reflect.SelectCase, 0, n)
    for i := 0; i < n; i++ {
        sc = append(sc, reflect.SelectCase{
            Chan: ls.Index(i),
            Dir:  reflect.SelectRecv,
        })
    }
    sc = append(sc, reflect.SelectCase{
        Dir:  reflect.SelectDefault,
    })
    i, v, ok := reflect.Select(sc)
    if !ok {
        return -1
    }
    reflect.ValueOf(dst).Elem().Set(v)
    return i
}

func main() {
    chs := make([]chan int, 10)
    for i := 0; i < len(chs); i++ {
        chs[i] = make(chan int, 1)
    }

    println("send to:", send(chs, 123))
    // Output: send to 7

    var v int
    println("recv from:", recv(chs, &v))
    // Output: recv from 7

    println(v)
    // Output: 123
}
```

如果用正常代码，上面的逻辑怎么实现？要知道数组`chs`是变长的，有可能增长，也可能缩短，这种情况下，用正常代码的`select`写法将会很难，比较简单的一种实现方式是：

```go
func init() {
    rand.Seed(time.Now().UnixNano())
}

// 注意重新洗牌时要记录原来的下标
func shuffle(chs []chan int) ([]chan int, []int) {
    tmp := make([]chan int, len(chs))
    idx := make([]int, len(chs))
    copy(tmp, chs)
    for i := 0; i < len(idx); i++ {
        idx[i] = i 
    }   
    rand.Shuffle(len(tmp), func(i, j int) {
        tmp[i], tmp[j] = tmp[j], tmp[i]
        idx[i], idx[j] = idx[j], idx[i]
    })
    return tmp, idx 
}

func send(chs []chan int, v int) int {
    chs, idx := shuffle(chs) // 每次都要重新洗牌
    for i, ch := range chs {
        select {
        case ch <- v:
            return idx[i]
        default:
        }
    }
    return -1
}

func recv(chs []chan int, v *int) int {
    chs, idx := shuffle(chs) // 每次都要重新洗牌
    for i, ch := range chs {
        select {
        case *v = <-ch:
            return idx[i]
        default:
        }
    }
    return -1
}
```

### 4.2 用反射写链表

**本例旨在：**
反射代码只是把普通代码完全展开，完整的写法。

一种经典的链表写法：

```go
type node struct {
    Data interface{}
    Next *node
}

type list *node

func makeList(ls *list, data ...interface{}) {
    for _, d := range data {
        *ls = &node{d, nil}
        ls = (*list)(&(*ls).Next)
    }
}

func main() {
    var ls list
    makeList(&ls, 1, true, "Hello world")
    for p := ls; p != nil; p = p.Next {
        fmt.Println(p.Data)
    }
    // Output:
    // 1
    // true
    // Hello world
}
```

请将上面代码`makeList`函数用反射实现：

```go
func makeList(ls interface{}, data ...interface{}) {
    p := reflect.ValueOf(ls).Elem()
    typ := p.Type().Elem()
    for _, d := range data {
        e := reflect.New(typ)
        e.Elem().FieldByName("Data").Set(reflect.ValueOf(d))
        p.Set(e)
        p = e.Elem().FieldByName("Next")
    }
}
```

### 4.3 钻空子

看下面这段代码：

```go
var t struct{ a int }
println(reflect.ValueOf(&t).Elem().Field(0).CanAddr())
// Output: true
```

看到这的时候，是不是有人动了歪心思：如果`CanAddr`，那是不是说可以通过它使未导出的字段可写？很不幸，官方告诉你：不行。

```go
var t struct{ a int }
v := reflect.ValueOf(&t).Elem().Field(0)
println(v.CanAddr())              // Output: true
println(v.Addr().Elem().CanSet()) // Output: false
println(v.Addr().CanInterface())  // Output: false
```

官方使未导出字段可读已经是突破下限的仁慈了：

```go
var t struct{ a int }
t.a = 123
println(reflect.ValueOf(&t).Elem().Field(0).Int())
// Output: 123
```

如果你就是想写未导出的字段，能不能做到呢？参考前面`unsafe`。

**最后善意的提醒：**
用`unsafe`修改未导出字段，这种做法将破坏原有逻辑，十分危险！

```go
type Buffer struct {
    buf       []byte
    off       int
    bootstrap [64]byte
    lastRead  uint8
}

buf := bytes.NewBuffer(nil)
buf.WriteString("1234567890")
println(buf.String()) // Output: 1234567890
(*Buffer)(unsafe.Pointer(buf)).off = 5
println(buf.String()) // Output: 67890
```

如果是接口，记得要先用`iface`过渡一下：

```go
type iface struct {
    typ unsafe.Pointer
    val unsafe.Pointer
}

type digest struct {
    h   [5]uint32
    x   [64]byte
    nx  int 
    len uint64
}

hs := sha1.New()
hs.Write([]byte("1234567890"))
d := (*digest)((*iface)(unsafe.Pointer(&hs)).val)
println(d.len) // Output: 10
```

### 4.4 小结

结合上述例子来看，反射其实没有什么神秘之处，反射代码和普通代码在逻辑上是完全一致的，唯一不太一样的点是：`type`，普通代码中的类型是直接声明出来的，反射代码中的类型需要自己推导。

---

## 5 结语

如果您已经看懂了本文所涉及的一些案例，恭喜您，反射对您已经不再神秘。反射已经由一把重剑巨阙化为灵巧翻飞的瑞士军刀。

关于反射的例子还有很多，但在这里不再过多涉及，因为大多例子和本文例子有异曲同工之妙。

在本文中，您看到的反射代码，都相当的简单，最重要的前提在于：我们对类型的限制。抛开大而全的理念，去做小而美，将更容易发挥反射的魅力。

不知道您有没有注意到，在案例中，反射和非反射代码，两种写法是混合交叉着在用的。这样做的好处在于，非反射代码能在一定程度上限制反射的使用，并保证反射代码的正确性。

大家会看到，我只是给出了实现方案，而没有给出优化方案，相信已经理解了本文内容的您，优化方案将很容易得到。

