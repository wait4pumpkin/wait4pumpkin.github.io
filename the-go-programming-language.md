# Go语言圣经

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


## TODO

+ range返回值or引用
+ range省略参数
+ 变量生命周期
