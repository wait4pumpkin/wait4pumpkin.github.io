---
layout: post
title: "The Go Programming Language"
categories: go
tags:
  - notes
---

## 程序结构

### 命名

关键字不能用于自定义名字，只能在特定语法结构中使用。内部预先定义的名字并不是关键字（包括builtin常量、类型、函数），可以再定义。

名字的开头字母的大小写决定了名字在包外的可见性。如果一个名字是大写字母开头的（必须是在函数外部定义的包级名字；包级函数名本身也是包级名字），那么它将是导出的，也就是说可以被外部的包访问。

驼峰命名，缩写使用全大写（ASCII、HTML）。

### 声明

有四种类型的声明语句：`var`、`const`、`type`和`func`。

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

一个原生的字符串面值形式是`` `...` ``，把`"`改为`` ` ``。原生字符串不进行转义，保留换行。

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

### 数组

数组长度是固定的，和slice的区别。默认进行初始化。

```go
// … 根据初值推导size
q := […]int{1, 2, 3}
```

长度是类型的一部分，长度不同就是不同类型。数组可以直接进行比较。

字面量初始化中，初始化索引的顺序不重要。未指定初始值的元素将用零值初始化。

```go
r := [...]int{99: -1}  // 大小是100
```

数组本身是值类型，指针要显示写出

```go
func zero(ptr *[32]byte) {
    *ptr = [32]byte{}
}
```

### Slice

没有固定长度的数组，相当于数组的引用。

```go
s := []int{0, 1, 2, 3, 4, 5}  // 区别在于没有长度
```

一个slice由三个部分构成：指针、长度和容量。内置的len和cap函数分别返回slice的长度和容量。指针指向slice第一个起始元素（不是数组），长度是slice元素数量，容量是slice开始位置到数组结尾的长度。

slice可以共享底层数据，引用区域可以重叠。

slice的切片操作`s[i:j]`，j可以大于`len(s)`扩大slice范围（不能超过cap(s)），索引范围必须在`len(s)`内。

slice不能直接用`==`比较（可以与`nil`比较）。不支持是因为slice可以引用自身，若只做引用相等检测，就会和数组行为不一致，所以直接禁止。

除了文档已经明确说明的地方，所有的Go语言函数应该以相同的方式对待nil值的slice和0长度的slice。

```go
var s []int    // len(s) == 0, s == nil
s = nil        // len(s) == 0, s == nil
s = []int(nil) // len(s) == 0, s == nil
s = []int{}    // len(s) == 0, s != nil
```

用`make`构造slice

```go
make([]T, len)
make([]T, len, cap) // same as make([]T, cap)[:len]
```

#### append函数

`append`可以往slice追加元素（多个或者slice），底层有可能重新分配内存空间，所以要用返回值重新赋值。

```go
var x []int
x = append(x, 4, 5, 6)
x = append(x, x...)  // append the slice x

runes = append(runes, r)
```

#### Slice内存技巧

利用slice共享内存的特性，避免内存分配

```go
func nonempty(strings []string) []string {
    i := 0
    for _, s := range strings {
        if s != "" {
            strings[i] = s
            i++
        }
    }
    return strings[:i]
}

func nonempty2(strings []string) []string {
    out := strings[:0]  // zero-length slice of original
    for _, s := range strings {
        if s != "" {
            out = append(out, s)
        }
    }
    return out
}

data := []string{"one", "", "three"}
data = nonempty(data)
```

模拟栈

```go
stack = append(stack, v)  // push v
top := stack[len(stack)-1]  // top of stack
stack = stack[:len(stack)-1]  // pop
```

删除元素（保持顺序）

```go
func remove(slice []int, i int) []int {
    copy(slice[i:], slice[i+1:])
    return slice[:len(slice)-1]
}

func main() {
    s := []int{5, 6, 7, 8, 9}
    fmt.Println(remove(s, 2)) // "[5 6 8 9]"
}
```

`swap`删除

```go
func remove(slice []int, i int) []int {
    slice[i] = slice[len(slice)-1]
    return slice[:len(slice)-1]
}

func main() {
    s := []int{5, 6, 7, 8, 9}
    fmt.Println(remove(s, 2)) // "[5 6 9 8]
}
```

### Map

key要支持`==`操作。可以用`make`创建或者字面量初始化。

```go
ages := make(map[string]int)
ages := map[string]int{}
```

即使元素不存在操作也是安全的。用下标访问不存在的key返回value的零值。

```go
ages["alice"] = 32
delete(ages, "alice")
```

如果要检测是否存在

```go
if age, ok := ages["bob"]; !ok { /* ... */ }
```

元素被禁止取址，因为内存可能会重分配（如果有这种需求应该存指针）。

```go
_ = &ages["bob"]  // compile error: cannot take address of map element
```

迭代顺序是随机的。map上的大部分操作，包括查找、删除、len和range循环都可以安全工作在nil值的map上，它们的行为和一个空的map类似（除了存入元素会panic）。

map也不能进行`==`比较，自行实现时要留意存在的时候才能用下标取值进行比较。

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

其实就是C++模式，非指针模式每次调用方法都是复制一个对象，指针就不复制。无论是调用指针对象方法或者调用非指针对象方法，都可以直接用`.`，编译器会自动补全取地址`&`或者取值`*`操作（前提是必须是一个变量）。

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

## 接口

接口是隐式实现的（描述方法集合），只要实现了接口约定的方法就是实现了该接口，不需要显式声明。

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}

func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
func Printf(format string, args ...interface{}) (int, error) {
    return Fprintf(os.Stdout, format, args...)
}
func Sprintf(format string, args ...interface{}) string {
    var buf bytes.Buffer
    Fprintf(&buf, format, args...)
    return buf.String()
}
```

### 接口类型

接口可以通过组合内嵌

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

### 实现接口的条件

要留意方法是定义在对象还是对象指针上。

```go
// String定义在对象指针上
var _ fmt.Stringer = &s  // OK
var _ fmt.Stringer = s   // compile error: IntSet lacks String method
```

`interface{}`空接口，可以接收任意类型（但无法进行任何操作）。

### 接口值

接口值可以使用＝＝和！＝来进行比较。两个接口值相等仅当它们都是nil值或者它们的动态类型相同并且动态值也根据这个动态类型的＝＝操作相等。但是，如果动态类型相同，但类型本身不支持比较，就会panic。

```go
var x interface{} = []int{1, 2, 3}
fmt.Println(x == x)  // panic: comparing uncomparable type []int
```

可以包接口抽象为两个指针，分别指向类型和值。特别要说明的是，包含`nil`的接口和`nil`接口是不同的。因为包含`nil`的接口，类型是接口，而`nil`类型空，类型不同就不同，并没有判断值。

```go
func main() {
    var buf *bytes.Buffer
    f(buf)  // NOTE: subtly incorrect!
}

// If out is non-nil, output will be written to it.
func f(out io.Writer) {
    if out != nil {
        out.Write([]byte("done!\n"))  // Panic
    }
}
```

### 类型断言

类型断言`x.(T)`，是对接口的操作，把接口转换为具体类型或者另一个接口。`nil`无论什么断言都会失败。

```go
var w io.Writer
w = os.Stdout
f := w.(*os.File)      // success: f == os.Stdout
c := w.(*bytes.Buffer) // panic: interface holds *os.File, not *bytes.Buffer
```

如果不希望panic

```go
if f, ok := w.(*os.File); ok {
    // ...use f...
}
```

### 类型分支

type switch，`.(type)`只能用于`switch`

```go
switch x.(type) {
    case nil:       // ...
    case int, uint: // ...
    case bool:      // ...
    case string:    // ...
    default:        // ...
}
```

## Goroutines和Channels

### Goroutines

当一个程序启动时，其主函数即在一个单独的goroutine中运行（main goroutine）。新的goroutine会用go语句来创建。除了从主函数退出或者直接终止程序之外，没有其它的编程方法能够让一个goroutine来打断另一个的执行。

### Channels

一个goroutine通过channel给另一个goroutine发送值信息。每个channel都有一个特殊的类型，也就是channels可发送数据的类型。

构造

```go
ch := make(chan int) // ch has type 'chan int'
```

和map类似，channel也对应一个make创建的底层数据结构的引用。当我们复制一个channel或用于函数参数传递时，我们只是拷贝了一个channel引用，因此调用者和被调用者将引用同一个channel对象。两个相同类型的channel可以使用==运算符比较。如果两个channel引用的是相同的对象，那么比较的结果为真。

发送和接收两个操作都使用`<-`运算符。一个不使用接收结果的接收操作也是合法的。

```go
ch <- x  // a send statement
x = <-ch // a receive expression in an assignment statement
<-ch     // a receive statement; result is discarded
```

Channel支持close操作，随后对基于该channel的任何发送操作都会panic。对一个已经被close过的channel进行接收操作依然可以接受到之前已经成功发送的数据；如果channel中已经没有数据的话将产生一个零值的数据。

```go
close(ch)
```

默认构造的channel是无缓存，可以指定缓存容量

```go
ch = make(chan int)    // unbuffered channel
ch = make(chan int, 0) // unbuffered channel
ch = make(chan int, 3) // buffered channel with capacity 3
```

### 不带缓存的Channels

一个基于无缓存Channels的发送操作将导致发送者goroutine阻塞，直到另一个goroutine在相同的Channels上执行接收操作。反之，如果接收操作先发生，那么接收者goroutine也将阻塞，直到有另一个goroutine在相同的Channels上执行发送操作。

### 串联的Channels（Pipeline）

判断channel是否关闭

```go
x, ok := <-naturals
```

用`range`，在channel关闭且没有数据时就会跳出

```go
for x := range naturals {
    squares <- x * x
}
```

只有当需要告诉接收者goroutine，所有的数据已经全部发送时才需要关闭channel。不管一个channel是否被关闭，当它没有被引用时将会被Go语言的垃圾自动回收器回收。

### 单方向的Channel

类型`chan<- int`表示一个只发送int的channel，只能发送不能接收。类型`<-chan int`表示一个只接收int的channel，只能接收不能发送。

因为关闭操作只用于断言不再向channel发送新的数据，所以只有在发送者所在的goroutine才会调用close函数。对一个只接收的channel调用close会编译错误。

任何双向channel向单向channel变量的赋值操作都将导致该隐式转换。不支持反向转换。

### 带缓存的Channels

如果内部缓存队列是满的，那么发送操作将阻塞直到因另一个goroutine执行接收操作而释放了新的队列空间。如果channel是空的，接收操作将阻塞直到有另一个goroutine执行发送操作而向队列插入元素。

`cap`和`len`可以获取channel的容量和有效元素个数。

如果goroutine阻塞在channel发送，而一直没有goroutine从channel接收，发送的goroutine就会泄露。和垃圾变量不同，泄漏的goroutines并不会被自动回收，因此确保每个不再需要的goroutine能正常退出是重要的。

```go
func mirroredQuery() string {
    responses := make(chan string, 3)  // 如果使用无缓冲，就会造成泄露
    go func() { responses <- request("asia.gopl.io") }()
    go func() { responses <- request("europe.gopl.io") }()
    go func() { responses <- request("americas.gopl.io") }()
    return <-responses // return the quickest response
}
```

### 并发的循环

用goroutine实现循环的并发

```go
func makeThumbnails3(filenames []string) {
    ch := make(chan struct{})
    for _, f := range filenames {
        go func(f string) {
            thumbnail.ImageFile(f) // NOTE: ignoring errors
            ch <- struct{}{}
        }(f)
        // f是复制了一份，如果直接capture是错误的
        // go func() {
        //     thumbnail.ImageFile(f) // NOTE: incorrect!
        //     // ...
        // }()
    }
    // Wait for goroutines to complete.
    for range filenames {
        <-ch
    }
}
```

如果考虑错误处理

```go
func makeThumbnails4(filenames []string) error {
    errors := make(chan error)

    for _, f := range filenames {
        go func(f string) {
            _, err := thumbnail.ImageFile(f)
            errors <- err
        }(f)
    }

    for range filenames {
        // 潜在问题，出错return时，发送的goroutine泄露。简单的解决方案是channel建立足够大的缓冲区，又或者多起一个goroutine负责消耗完channel内容
        if err := <-errors; err != nil {
            return err // NOTE: incorrect: goroutine leak!
        }
    }

    return nil
}
```

通过`sync.WaitGroup`等待完成

```go
func makeThumbnails6(filenames <-chan string) int64 {
    sizes := make(chan int64)
    var wg sync.WaitGroup // number of working goroutines
    for f := range filenames {
        wg.Add(1) // 要在goroutine外，否则closer可能就直接结束了
        // worker
        go func(f string) {
            defer wg.Done() // 保证一定-1
            thumb, err := thumbnail.ImageFile(f)
            if err != nil {
                log.Println(err)
                return
            }
            info, _ := os.Stat(thumb) // OK to ignore error
            sizes <- info.Size()
        }(f)
    }

    // closer
    go func() {
        wg.Wait()
        close(sizes) // close的时候不会再有发送
    }()

    var total int64
    for size := range sizes {
        total += size
    }
    return total
}
```

### 示例: 并发的Web爬虫

```go
func main() {
    worklist := make(chan []string)

    // Start with the command-line arguments.
    go func() { worklist <- os.Args[1:] }()

    // Crawl the web concurrently.
    seen := make(map[string]bool)
    for list := range worklist {
        for _, link := range list {
            if !seen[link] {
                seen[link] = true
                // 开一个goroutine是为了避免没有读操作而一直阻塞
                go func(link string) {
                    worklist <- crawl(link)
                }(link)
                // link要复制，不能直接capture
            }
        }
    }
}
```

通过buffered channel限制并发上限

```go
// 令牌限制
var tokens = make(chan struct{}, 20)

func crawl(url string) []string {
    tokens <- struct{}{} // acquire a token
    list, err := links.Extract(url)
    <-tokens // release the token
    if err != nil {
        log.Print(err)
    }
    return list
}
```

为了能让程序正常退出，记录要获取的链接数

```go
func main() {
    worklist := make(chan []string)
    var n int // number of pending sends to worklist

    // Start with the command-line arguments.
    n++
    go func() { worklist <- os.Args[1:] }()

    // Crawl the web concurrently.
    seen := make(map[string]bool)

    for ; n > 0; n-- {
        list := <-worklist
        for _, link := range list {
            if !seen[link] {
                seen[link] = true
                n++
                go func(link string) {
                    worklist <- crawl(link)
                }(link)
            }
        }
    }
}
```

又或者通过常驻的goroutine限制并发

```go
func main() {
    worklist := make(chan []string)  // lists of URLs, may have duplicates
    unseenLinks := make(chan string) // de-duplicated URLs

    // Add command-line arguments to worklist.
    go func() { worklist <- os.Args[1:] }()

    // Create 20 crawler goroutines to fetch each unseen link.
    for i := 0; i < 20; i++ {
        go func() {
            for link := range unseenLinks {
                foundLinks := crawl(link)
                go func() { worklist <- foundLinks }()
            }
        }()
    }

    // The main goroutine de-duplicates worklist items
    // and sends the unseen ones to the crawlers.
    seen := make(map[string]bool)
    for list := range worklist {
        for _, link := range list {
            if !seen[link] {
                seen[link] = true
                unseenLinks <- link
            }
        }
    }
}
```

### 基于select的多路复用

每一个case代表一个通信操作(在某个channel上进行发送或者接收)并且会包含一些语句组成的一个语句块。select会有一个default来设置当其它的操作都不能够马上被处理时程序需要执行哪些逻辑（相当于非阻塞）。

```go
select {
case <-ch1:
    // ...
case x := <-ch2:
    // ...use x...
case ch3 <- y:
    // ...
default:
    // ...
}
```

一个没有任何case的select语句写作select{}，会永远地等待下去。如果多个case同时就绪时，select会随机地选择一个执行，这样来保证每一个channel都有平等的被select的机会。

```go
// 如果增大buffer，执行就是随机的
ch := make(chan int, 1)
for i := 0; i < 10; i++ {
    select {
    case x := <-ch:
        fmt.Println(x) // "0" "2" "4" "6" "8"
    case ch <- i:
    }
}
```

channel的零值是nil。因为对一个nil的channel发送和接收操作会永远阻塞，在select语句中操作nil的channel永远都不会被select到。

### 并发的退出

用channel进行广播，退出之前要保证某些必要的channel被清空，避免其他goroutine一直阻塞导致泄露。

```go
var done = make(chan struct{})

func cancelled() bool {
    select {
    case <-done:
        return true
    default:
        return false
    }
}
```

## 基于共享变量的并发

尽量不要使用共享数据来通信，使用通信来共享数据（channel）。

### sync.Mutex互斥锁

go里没有重入锁

### sync.RWMutex读写锁

```go
var mu sync.RWMutex
var balance int
func Balance() int {
    mu.RLock() // readers lock
    defer mu.RUnlock()
    return balance
}
```



## 反射

### 为何需要反射?


## 底层编程

### unsafe.Sizeof, Alignof 和 Offsetof

基本功能和C++接近

unsafe.Sizeof函数返回操作数在内存中的字节大小，参数可以是任意类型的表达式，但是它并不会对表达式进行求值。

+ bool：1个字节
+ int, uint, uintptr：1个机器字
+ *T：1个机器字
+ string：2个机器字（data、len）
+ []T：3个机器字（data、len、cap）
+ map：1个机器字
+ func：1个机器字
+ chan：1个机器字
+ interface：2个机器字（type、value）

Go语言的规范并没有要求一个字段的声明顺序和内存中的顺序是一致的，所以理论上一个编译器可以随意地重新排列每个字段的内存位置，但目前实现并没有这么做。

这几个方法调用事实上是安全的

### unsafe.Pointer

unsafe.Pointer是特别定义的一种指针类型（类似`void *`），它可以包含任意类型变量的地址。我们不可以直接通过`*p`来取值，因为缺失具体类型。unsafe.Pointer指针可以比较，并且支持和nil常量比较判断是否为空指针。

普通的`*T`类型指针可以与unsafe.Pointer类型指针互相转换，且被转回普通的指针类型并不需要和原始的`*T`类型相同。

unsafe.Pointer指针可以被转化为uintptr类型，然后进行指针运算。

```go
var x struct {
    a bool
    b int16
    c []int
}

// 和 pb := &x.b 等价
pb := (*int16)(unsafe.Pointer(
    uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)))
*pb = 42
fmt.Println(x.b) // "42"
```

## TODO

+ range返回值or引用
+ range省略参数
+ 变量生命周期
+ 反射实现
+ 接口实现（多态）
+ nil的比较，nil的类型
+ gc什么时候执行，一个语句内可能执行gc吗

+ 3.6 常量
+ 4.4 结构体
+ 6.6 封装

+ for range修改map
+ 没有重载
+ 类型断言消耗
+ goroutine并发数、调度，什么时候会让出
+ channel阻塞怎么调度唤醒
+ channel怎么保证并发安全
+ channel显式close or 等待gc
+ 有没有引用计数
+ empty struct大小
+ 线程安全容器
+ Mutex和channel消耗对比

http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/

The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next. If map entries that have not yet been reached are removed during iteration, the corresponding iteration values will not be produced. If map entries are created during iteration, that entry may be produced during the iteration or may be skipped. The choice may vary for each entry created and from one iteration to the next. If the map is nil, the number of iterations is 0.
