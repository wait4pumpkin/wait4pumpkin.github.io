# Go语言圣经

## 基础数据类型

### 整型

int和uint，大小相同，可能为32或64bit。

Unicode字符rune类型是和int32等价的类型，通常用于表示一个Unicode码点。

byte也是uint8类型的等价类型，byte类型一般用于强调数值是一个原始的数据而不是一个小的整数。

无符号的整数类型uintptr，没有指定具体的bit大小但是足以容纳指针。

int和int32也是不同的类型，即使int的大小也是32bit，在需要将int当作int32类型的地方需要一个显式的类型转换操作。

有符号整数采用2的补码形式表示。

取模运算符%仅用于整数间的运算。%取模运算符的符号和被取模数的符号总是一致的（a % b，与a一致）。

位运算`&`、`|`、`^`、`&^`（位清空，x &^ y，按y标记位清空x），不区分符号。

算术和逻辑运算的二元操作中必须是相同的类型，类型不会自动提升。向下强制转换，截断行为依赖于具体实现。

### 浮点数

函数math.IsNaN用于测试一个数是否是非数NaN。直接用math.NaN进行比较是有问题的，因为NaN和任何数都是不相等的（在浮点数中，NaN、正无穷大和负无穷大的表示都不是唯一的）。

浮点数相等的比较也要注意精度问题。

### 复数

两种精度的复数类型：complex64和complex128，分别对应float32和float64两种浮点数精度。内置的complex函数用于构建复数，内建的real和imag函数分别返回复数的实部和虚部（传入的实部和虚部大小要相同，如果float32则返回complex64）。

### 布尔值

`&&`的优先级比`||`高

```go
// 可以不加括号
if 'a' <= c && c <= 'z' ||
    'A' <= c && c <= 'Z' ||
    '0' <= c && c <= '9' {
    // ...ASCII letter or digit...
}
```

布尔值并不会隐式转换为数字值0或1，反之亦然。

### 字符串

一个字符串是一个不可改变的字节序列。内置的`len`函数可以返回一个字符串中的字节数目，索引操作`s[i]`返回第i个字节的字节值（都是以字节为单位，不是码点）。

不变性说明两个字符串共享相同的底层数据的话也是安全的。因此，复制任何长度的字符串，或者取子字符串切片，消耗也很小。

Go语言源文件总是用UTF8编码，并且Go语言的文本字符串也以UTF8编码的方式处理。

一个原生的字符串面值形式是`` `...` ``，把`"`改为` ` `。原生字符串不进行转义，保留换行。

#### Unicode

Unicode码点对应Go语言中的rune整数类型（rune是int32等价类型）。

range循环在处理字符串的时候，会自动隐式解码UTF8字符串（并不是字节遍历）。

```go
for i, r := range "Hello, 世界" {
    fmt.Printf("%d\t%q\t%d\n", i, r, r)
}
```

转换为`rune`序列进行处理

```go
s := "プログラム"
fmt.Printf("% x\n", s) // "e3 83 97 e3 83 ad e3 82 b0 e3 83 a9 e3 83 a0"
r := []rune(s)
fmt.Printf("%x\n", r)  // "[30d7 30ed 30b0 30e9 30e0]"
```

## 复合数据类型




## 函数

### 函数声明

实参通过值的方式传递，因此函数的形参是实参的拷贝。对形参进行修改不会影响实参。但是，如果实参包括引用类型，如指针，slice(切片)、map、function、channel等类型，实参可能会由于函数的间接引用被修改。

没有函数体的函数声明，这表示该函数不是以Go实现的。

### 递归

可变栈，动态分配在堆上，要制造栈溢出比固定栈要难（但也不是不行，毕竟栈大小还是有上限）

### 多返回值

如果一个函数将所有的返回值都显式地写出变量名，那么该函数的return语句可以省略操作数。这称之为bare return。

```go
// CountWordsAndImages does an HTTP GET request for the HTML
// document url and returns the number of words and images in it.
func CountWordsAndImages(url string) (words, images int, err error) {
    resp, err := http.Get(url)
    if err != nil {
        return
    }
    doc, err := html.Parse(resp.Body)
    resp.Body.Close()
    if err != nil {
        err = fmt.Errorf("parsing HTML: %s", err)
        return
    }
    words, images = countWordsAndImages(doc)
    return  // 不可去掉
}
func countWordsAndImages(n *html.Node) (words, images int) { /* ... */ }
```

### 错误

通常，当函数返回non-nil的error时，其他的返回值是未定义的(undefined),这些未定义的返回值应该被忽略。然而，有少部分函数在发生错误时，仍然会返回一些有用的返回值。

由于错误信息经常是以链式组合在一起的，所以错误信息中应避免大写和换行符。

```go
doc, err := html.Parse(resp.Body)
resp.Body.Close()
if err != nil {
    return nil, fmt.Errorf("parsing %s as HTML: %v", url,err)
}
```

### 函数值

函数值之间是不可比较的（equal也不行），也不能用函数值作为map的key

### 匿名函数

具名函数只能在包级语法块中被声明，其他地方可以用匿名函数。

匿名函数实现递归

```go
visitAll := func(items []string) {
    // ...
    visitAll(m[item]) // compile error: undefined: visitAll
    // ...
}

var visitAll func(items []string)
visitAll = func(items []string) {
    // ...
    visitAll(m[item])
    // ...
}
```

注意闭包捕获变量的作用域

```go
// d在是迭代过程中共享的，所以必须要复制一份
var rmdirs []func()
for _, d := range tempDirs() {
    dir := d // NOTE: necessary!
    os.MkdirAll(dir, 0755) // creates parent directories too
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dir)
    })
}
// ...do some work…
for _, rmdir := range rmdirs {
    rmdir() // clean up
}
```

### 可变参数

```go
func sum(vals ...int) int {
    total := 0
    for _, val := range vals {
        total += val
    }
    return total
}
```

调用者隐式的创建一个数组，并将原始参数复制到数组中，再把数组的一个slice作为参数传给被调函数（但和跟接收slice的函数是不同类型）

```go
sum(1, 2, 3, 4)

values := []int{1, 2, 3, 4}
sum(values...)
```

### Deferred函数

+ defer是在包含该语句的函数执行完成后执行，而不是enclosing brackets；
+ defer事实上是执行了一次，把函数引用和参数进行了复制；
+ 即使是panic，也会执行；
+ defer的调用也有消耗，hot path应该避免使用；
+ 多条defer执行顺序与声明顺序相反。

```go
// work全部执行完才会执行文件的关闭
func work() {
    for _, filename := range filenames {
        f, err := os.Open(filename)
        if err != nil {
            return err
        }
        defer f.Close() // NOTE: risky; could run out of file
        // ...process f…
    }
}

// 包匿名函数解决
func work() {
    for _, filename := range filenames {
        func () {
            f, err := os.Open(filename)
            if err != nil {
                return err
            }
            defer f.Close()
            // ...process f…
        }()
    }
}

```

### Panic异常

在Go的panic机制中，延迟函数的调用在释放堆栈信息之前。

```go
func main() {
    defer printStack()
    f(3)
}

func printStack() {
    var buf [4096]byte
    n := runtime.Stack(buf[:], false)
    os.Stdout.Write(buf[:n])
}

func f(x int) {
    fmt.Printf("f(%d)\n", x+0/x) // panics if x == 0
    defer fmt.Printf("defer %d\n", x)
    f(x - 1)
}
```

### Recover捕获异常

如果在deferred函数中调用了内置函数recover，并且定义该defer语句的函数发生了panic异常，recover会使程序从panic中恢复，并返回panic value。导致panic异常的函数不会继续运行，但能正常返回。在未发生panic时调用recover，recover会返回nil。

```go
func Parse(input string) (s *Syntax, err error) {
    defer func() {
        if p := recover(); p != nil {
            err = fmt.Errorf("internal error: %v", p)
        }
    }()
    // ...parser...
}
```

## 方法

### 方法声明

在函数声明时，在其名字之前放上一个变量，即是一个方法。附加的参数，叫做方法的接收器（receiver）。

方法可以被声明到任意类型（slice、字符串、数值都可以），只要不是一个指针或者一个interface。

```go
type Path []Point
func (path Path) Distance() float64 {
    // ...
}
```

### 基于指针对象的方法

其实就是C++模式，非指针模式每次调用方法都是复制一个对象，指针就不复制。无论是调用指针对象方法或者调用非指针对象方法，都可以直接用`.`，编译器会自动补全取地址`&`或者取值`*`操作。

自动补全模式仅限于变量，如果无法取到地址（例如临时变量），就无法调用

```go
Point{1, 2}.ScaleBy(2)  // compile error: can't take address of Point literal
```

一般来说，用了指针对象方法，所有方法都用指针，不会混用。

只有类型和对应的指针会出现在接收器声明。如果类型本身是一个指针，会造成歧义，所以直接禁止。

```go
type P *int
func (P) f() { /* ... */ }  // compile error: invalid receiver type
```

`nil`也是一个合法的接收器类型，如果方法允许接收最好在注释内声明

```go
m := url.Values{"lang": {"en"}} // direct construction
m.Add("item", "1")

m = nil
fmt.Println(m.Get("item"))  // ""，但是nil.Get("item")编译错，因为无法判断类型
m.Add("item", "3")  // panic: assignment to entry in nil map
```

### 通过嵌入结构体来扩展类型

Go没有继承，是通过组合的方式。事实上就是一个语法上的简写。搜索属性的时候会优先选择自身定义的具名属性和方法，然后再搜索匿名的嵌入结构体（不一定是定义在开头）。

匿名结构体可以有多个，如果搜索过程结果存在二义性，则编译错误。

```go
type Point struct{ X, Y float64 }

type ColoredPoint struct {
    Point
    Color color.RGBA
}

var cp ColoredPoint
cp.X = 1
fmt.Println(cp.Point.X) // "1"

// 相当于编译期自动生成了方法
func (p ColoredPoint) Distance(q Point) float64 {
    return p.Point.Distance(q)
}
```

匿名字段也可以是指针，多个对象可以共享这个匿名对象

```go
type ColoredPoint struct {
    *Point
    Color color.RGBA
}

p := ColoredPoint{&Point{1, 1}, red}
q := ColoredPoint{&Point{5, 4}, blue}
q.Point = p.Point
p.ScaleBy(2)
fmt.Println(*p.Point, *q.Point) // "{2 2} {2 2}"
```

使用匿名结构体的例子

```go
var cache = struct {
    sync.Mutex
    mapping map[string]string
}{
    mapping: make(map[string]string),
}

func Lookup(key string) string {
    cache.Lock()
    v := cache.mapping[key]
    cache.Unlock()
    return v
}
```

### 方法值和方法表达式

这就相当于Python的bound method和unbound method。

```go
p := Point{1, 2}
q := Point{4, 6}

distanceFromP := p.Distance        // method value
fmt.Println(distanceFromP(q))      // "5"

type Rocket struct { /* ... */ }
func (r *Rocket) Launch() { /* ... */ }
r := new(Rocket)
time.AfterFunc(10 * time.Second, r.Launch)  // 直接用bound method

// unbound method，接收器变成第一个参数
distance := Point.Distance   // method expression
fmt.Println(distance(p, q))  // "5"
fmt.Printf("%T\n", distance) // "func(Point, Point) float64"
```

### 封装




## TODO

+ range返回值or引用
+ range省略参数
+ 变量生命周期

+ 3.6常量
+ 6.6封装

+ for range修改map

The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next. If map entries that have not yet been reached are removed during iteration, the corresponding iteration values will not be produced. If map entries are created during iteration, that entry may be produced during the iteration or may be skipped. The choice may vary for each entry created and from one iteration to the next. If the map is nil, the number of iterations is 0.
