# 类型推导

## 条款1：理解模板类型推导

```c++
template<typename T>
void f(ParamType param);  // ParamType可能带有修饰词

f(expr);  // 从expr推导T和ParamType类型
```

### 情况1：ParamType是指针或引用，但不是万能引用

若expr是引用类型，则忽略引用部分，然后对expr的类型和ParamType的类型进行匹配，决定T的类型

```c++
template<typename T>
void f(T& param);

int x = 1;
const int cx = x;
const int& rx = x;

f(x);  // T是int，param是int&
f(cx);  // T是const int，param是const int&，const保留了
f(rx);  // T是const int，param是const int&，因为引用忽略了事实上和cx相同
```

```c++
template<typename T>
void f(const T& param);

int x = 1;
const int cx = x;
const int& rx = x;

f(x);  // T是int，param是const int&
f(cx);  // T是int，param是const int&，param有const，T不需要保留
f(rx);  // T是int，param是const int&，因为引用忽略了事实上和cx相同
```

```c++
template<typename T>
void f(T* param);

int x = 1;
const int *px = &x;

f(&x);  // T是int，param是int*
f(px);  // T是const int，param是const int*
```

### 情况2：ParamType是万能引用

+ 如果expr是左值，则T和ParamType均被推导为左值引用（模板推导中T被推导为引用的唯一情形）
+ 如果expr是右值，则适用情况1

```c++
template<typename T>
void f(T&& param);

int x = 1;
const int cx = x;
const int& rx = x;

f(x);  // T是int&，param是int&
f(cx);  // T是const int&，param是const int&
f(rx);  // T是const int&，param是const int&
f(1);  // T是int，param是int&&
```

### 情况3：ParamType既非指针也非引用

就是按值传递，引用部分被忽略，const和valatile也被忽略

```c++
template<typename T>
void f(T param);

int x = 1;
const int cx = x;
const int& rx = x;

f(x);  // T是int，param是int
f(cx);  // T是int，param是int，因为是复制，const不需要保留
f(rx);  // T是int，param是int

const char* const ptr = "abc";
f(ptr);  // T是const char*
```

数组实参很多语境下可以退化为首元素的指针，但引用可以推导为数组

```c++
const char name[] = "abc";

template<typename T>
void f(T param);

f(name);  // T是const char*

template<typename T>
void f(T& param);

f(name);  // T是const char[3]，param是const char (&)[13]

// 可以用于获取数组大小
template<typename T, std::size_t N>
constexpr std::size_t size(T (&)[N]) noexcept
{
  return N;
}
```

函数也可退化为函数指针

```c++
void func(int);

template<typename T>
void f(T param);

f(func);  // param类型是void (*)(int)

template<typename T>
void f(T& param);

f(func);  // param类型是void (&)(int)

// 在引用函数名但又没有调用该函数时，函数名将被自动解释为函数指针，用不用取址都可以达到效果
// 使用函数指针调用函数时，可以不需要加解引用
```


## 条款2：理解auto类型推导

一般情况下，auto类型推导和模板类型推导规则是完全相同的，auto可以看做是模板的T

```c++
auto x = 27;  // 情况3，T是int
const auto cx = x;  // 情况3，T是int
const auto& rx = x;  // 情况1，T是int

auto&& ref1 = x;  // 情况2，T是int&
auto&& ref2 = cx;  // 情况2，T是const int&
auto&& ref3 = 27;  // 情况2，T是int

// 数组和函数规则与模板推导相同
const char name[] = "abc";
auto arr1 = name;  // T是const char*
auto& arr2 = name;  // T是const char (&)[3]

void func(int);
auto func1 = func;  // T是void (*)(int)
auto& func2 = func;  // T是void (&)(int)
```

唯一不同的是大括号初始化，如果auto声明的初始化表达式是用大括号时，推导的类型就属于`std::initializer_list<T>`，包含了两次模板推导，auto和初始化列表。而模板是无法推导大括号初始化表达式。

```c++
// 四种初始化结果相同
int x1 = 1;
int x2(1);
int x3 = {1};
int x4{1};

// T是int
auto x1 = 1;
auto x2(1);

// T是std::initializer_list<int>，值是{1}
auto x3 = {1};
auto x4{1};

auto x5 = {1, 2, 3.0};  // ERROR，initializer_list的模板参数推导失败

template<typename T>
void f(T param);

auto x = {1, 2, 3};
f({1, 2, 3});  // ERROR

template<typename T>
void f(std::initializer_list<T>);

f({1, 2, 3});  // T是int
```

auto在返回值和参数中应用的是模板匹配，而不是auto匹配规则

```c++
auto create()
{
  return {1, 2, 3};  // ERROR
}

auto func = [](const auto& value) {};
func({1, 2, 3});  // ERROR
```


## 条款3：理解decltype

绝大部分情况下，decltype会得出表达式或者变量的类型而不做任何修改（就是不走模板匹配的规则）

decltype(auto)表示的是类型推导使用的decltype的规则

```c++
template<typename Container, typename Index>
// auto access(Container&& c, Index i) -> decltype(c[i])  // C++11没有问题
// auto应用的是模板匹配规则，如果存在引用会被丢弃，，因此要用decltype(auto)
decltype(auto) access(Container&& c, Index i)  // C++14
{
  return std::forward<Container>(c)[i];
}

Widget w;
const Widget& cw = w;

auto w1 = cw;  // 情况3，T是Widget
decltype(auto) w2 = cw;  // decltype推导规则，const Widget&
``

如果decltype应用在比单个变量名更复杂的左值表达式的话，得出的类型总是左值引用

```c++
int x = 0;
decltype(x)  // int
decltype((x))  // int&

decltype(auto) f()
{
  int x = 0;
  return (x);  // 返回int&，未定义行为
}
```


## 条款4：掌握查看类型推导结果的方法

编译期，尝试产生编译错，从错误信息中获取

```c++
template<typename T>  // 只声明不定义
class TD;  // Type Display

TD<decltype(x)> xType;  // 产生编译错
```

运行时输出，GUN和Clang编译期需要c++filt来转换（`c++filt -t xxx`）

```c++
std::cout << typeid(x).name() << std::endl;  // std::type_info::name
```

但该方法对const、引用等都不准确（标准允许，与函数模板按值传递性质相同）

```c++
std::vector<Widget> createVec();
const auto v = createVec();
f(&v[0]);

template<typename T>
void f(const T &param)
{
  std::cout << "T = " << typeid(T).name() << "\n";  // Widget const *
  std::cout << "param = " << typeid(param).name() << "\n";  // Widget const *
}

// T应该是Widget const *, param是Widget const * const &
// 按条款1，传递时去除了const和&
```

使用Boost的`TypeIndex`可精确显示类型

```c++
#include <boost/type_index.hpp>

template<typename T>
void f(const T &param)
{
  using boost::typeindex::type_id_with_cvr;

  std::cout << "T = " << type_id_with_cvr<T>().pretty_name() << "\n";  // Widget const *
  std::cout << "T = " << type_id_with_cvr<decltype(param)>().pretty_name() << "\n";  // Widget const * const &
}
```


# auto

## 条款5：优先选用auto，而非显示类型声明

```c++
// 用auto可以省却复杂声明
typename std::iterator_traites<It>::value_type v = *it;  // 不加typename认为是值

// 用auto可以表示只有编译器才掌握的类型
auto func =
  [](const std::unique_ptr<Widget> &p1, const std::unique_ptr<Widget> &p2)
  { return *p1 < *p2; }

// C++14支持lambda形参也用auto
auto func =
  [](const auto &p1, const auto &p2)
  { return *p1 < *p2; }

// 用std::function可以显式声明，但事实上存储着闭包的实例，内存性能都会受影响
std::function<
  bool(const std::unique_ptr<Widget> &, const std::unique_ptr<Widget> &)> func =
  [](const std::unique_ptr<Widget> &p1, const std::unique_ptr<Widget> &p2)
  { return *p1 < *p2; }

// 类型变化时不需要修改代码
std::vector<int> v;
unsigned sz = v.size();  // 可能出错
auto sz = v.size();  // 类型是std::vector<int>::size_type
zhengque
// 使用auto可以避免类型错误
std::unordered_map<std::string, int> m;
for (const auto &p : m) { /* ... */ }  // 能够保证类型正确
for (const std::pair<std::string, int> &p : m) { /* ... */ }  // 经典错误
for (std::pair<std::string, int> &p : m) { /* ... */ }  // 编译错
```

正确的类型应该是`std::pair<const std::string, int>`，上面的写法事实上包含了一次隐式转换，性能下降。从最后一个编译错可以更容易看出这个问题，隐式转换出来的是右值，非常量引用不能绑定右值，从错误可以发现类型错误。

基于范围的for

```c++
for ( 范围声明 : 范围表达式 ) 循环语句
```

等价于

```c++
{
  auto && __range = 范围表达式 ;
  for (auto __begin = 首表达式, __end = 尾表达式; __begin != __end; ++__begin)
  {
    范围声明 = *__begin;
    循环语句
  }
}
```

首表达式 与 尾表达式 定义如下：

+ 若 范围表达式 是数组类型表达式，则 首表达式 为`__range`而 尾表达式 为`(__range + __bound)`，其中`__bound`是数组的元素数目（若数组大小未知或拥有不完整类型，则程序不合法）
+ 若 范围表达式 是拥有名为`begin`以及`end`成员的类类型 C 的表达式（不管该成员的类型或可见性），则 首表达式 为`__range.begin()`且 尾表达式 为`__range.end()`；
+ 否则， 首表达式 为`begin(__range)`且 尾表达式 为`end(__range)``，通过参数依赖查找寻找（不进行非 ADL 查找）。


## 条款6：当auto推导的类型不符合要求时，使用带显式类型的初始化物习惯用法

隐形代理可能会导致auto推导的类型不满足需求

```c++
std::vector<bool> features(const Widget &w);

Widget w;
bool priority = features(w)[5];  // 隐式类型转换
processWidget(w, priority);  // correct

auto priority = features(w)[5];  // std::vector<bool>::reference
processWidget(w, priority);  // undefined

Matrix sum = m1 + m2 + m3 + m4;  // operator+的返回值同样是代理
```

`std::vector<bool>`做了特化，是按位存储的，`operator[]`返回的是代理对象，`std::vector<bool>::reference`。`features`返回的是临时对象，在调用`processWidget`时，`priority`内部持有的是悬垂指针，因此行为未定义。

因此对于代理对象可能需要显式的转换来确保推导的类型正确。

```c++
auto priority = static_cast<bool>(features(w)[5]);
auto sum = static_cast<Matrix>(m1 + m2 + m3 + m4);
```


# 转向现代C++

## 条款7：在创建对象时注意区分()和{}

多种初始化方式，等号并不等同于赋值

```c++
int x(0);
int y = 0;
int z { 0 };  // int z = { 0 };等同

Widget w1;  // 默认构造函数
Widget w2 = w1;  // 复制构造函数
w1 = w2;  // 赋值运算符
```

C++11引入统一初始化：单一的、可用于一切场合、表达一切意思的初始化，形式上就是大括号初始化。

指定容器初始内容

```c++
std::vector<int> v { 1, 3, 5 };  // v的初始内容为1,3,5
```

非静态成员默认初始化值，=也可以，小括号不行

```c++
class Widget
{
private:
  int x{ 0 };
  int y = 0;
  int z(0);  // ERROR
};
```

不可复制的对象，可以用大括号和小括号初始化。所谓不可复制，事实上就是`T(const T &) = delete`，最后一个用例包含了两步操作，第一步是通过转换构造函数生成临时对象，然后再调用复制构造函数。但真实调用中，复制构造函数是不会被调用的，因为C++的复制消除特性（copy elision）。

```c++
std::atomic<int> ai1 { 0 };
std::atomic<int> ai2(0);
std::atomic<int> ai3 = 0;  // ERROR
```

大括号初始化禁止narrowing conversion，小括号和=是不进行检查（C++支持的隐式转换）

```c++
double x, y, z;
int sum1{ x + y + z};  // ERROR
int sum2(x + y + z);
int sum3 = x + y + z;

// 浮点类型纯右值能隐式转换成任何整数类型的纯右值。截断小数部分，即舍弃小数部分。若结果不能适应到目标类型中，则行为未定义（即使在目标类型为无符号数时，也不应用模算术）
```

大括号初始化可以免疫most vexing parse（C++规定，任何能够解析为声明的都要解析为声明）

```c++
Widget w1(10);  // 调用构造函数
Widget w2();  // 解析语法，声明了一个函数，而不是变量
Widget w3{};  // 初始化
```

大括号初始化的缺点在于，如果存在构造函数声明任何一个具备std::initializer_list类型的形参，大括号初始化的调用会强烈优先选择该重载版本。

```c++
class Widget
{
public:
  Widget(int, bool);
  Widget(int, double);
};

Widget w1(10, true);  // 1
Widget w2{10, true};  // 1
Widget w3(10, 5.0);  // 2
Widget w4{10, 5.0};  // 2


class Widget
{
public:
  Widget(int, bool);
  Widget(int, double);
  Widget(std::initializer_list<long double>);
};

Widget w1(10, true);  // 1
Widget w2{10, true};  // 3，强制转为long double
Widget w3(10, 5.0);  // 2
Widget w4{10, 5.0};  // 3，强制转为long double
```

即使复制和移动构造函数也会受影响

```c++
class Widget
{
public:
  Widget(int, bool);
  Widget(int, double);
  Widget(std::initializer_list<long double>);

  operator float() const;  // 假如不能转换不会调3
};

Widget w5(w4);  // 复制构造函数
Widget w6{w4};  // 调用3，w4先被转换为float，再转换为long double
Widget w7(std::move(w4));  // 移动构造
Widget w8{std::move(w4)};  // 调用3，w4先被转换为float，再转换为long double
```

可能无法调用精确匹配的重载版本

```c++
class Widget
{
public:
  Widget(int, bool);
  Widget(int, double);
  Widget(std::initializer_list<bool>);
};

Widget w{10, 5.0};  // ERROR，存在转换方法，但大括号禁止narrow conversion
```

只有不存在任何转换方法时才退而检查普通重载决议

```c++
class Widget
{
public:
  Widget(int, bool);
  Widget(int, double);
  Widget(std::initializer_list<std::string>);
};

Widget w1(10, true);  // 调用1
Widget w2{10, true};  // 调用1
Widget w3(10, 5.0);  // 调用2
Widget w4{10, 5.0};  // 调用2
```

空的大括号，按语言规定是调用默认构造，而不是空的std::initializer_list，除非显式地表明

```c++
Widget w4({});
Widget w5{{}};
```

std::initializer_list对初始化重载函数的选择会影响到接口的设计，一般认为std::vector的接口设计为败笔（用小括号或者大括号会影响到调用的重载版本）

一般在开发过程中默认选用一种，在必要的情况下选用另外一种。

对模板开发来说选择更困难

```c++
template<typename T, typename... Ts>
void work(Ts&&... params)
{
  // 使用()或{}只有调用者才有决定权
  // 标准库内选择的是()（std::make_unique和std::make_shared），并在文档中说明
}

T local(std::forward<Ts>(params));
T local{std::forward<Ts>(params)};
```

另外一种解决方案是使用tag dispatching来定义要使用的行为，`tag`本质上是没有数据的对象。

```c++
namespace std{
  constexpr struct with_size_t{} with_size{};
  constexpr struct with_value_t{} with_value{};
  constexpr struct with_capacity_t{} with_capacity{};
}

std::vector<int> v(std::with_size, 10, std::with_value, 6);
```

如果是描述数据，更推荐的是用强类型（当然要权衡性能、通用性等因素）
```c++
class Circle
{
public:
    struct buildWithRadius{};
    struct buildWithDiameter{};

    explicit Circle(double radius, buildWithRadius);
    explicit Circle(double diameter, buildWithDiameter);
};

class Circle
{
public:
    explicit Circle(Radius radius);
    explicit Circle(Diameter diameter);
};
```

stl里面也大量使用

```c++
std::vector<int> v = { 1, 2, 3, 4, 5 };
auto it = v.begin();
std::advance(it, 3);

template <typename Iterator, typename Distance>
void advance_impl(Iterator& it, Distance n, forward_iterator_tag)
{
    while (--n >= 0)
        ++it;
}

template <typename Iterator, typename Distance>
void advance_impl(Iterator& it, Distance n, random_iterator_tag)
{
    it += n;
}
```


## 条款8：优先选用nullptr，而非0或NULL

0和NULL都不是指针类型，只有C++发现在只能使用指针的环境发现0或者NULL，才会解释为指针。

```c++
void f(int);
void f(bool);
void f(void*);

f(0);  // 调用f(int)
f(NULL);  // 可能编译不通过，一般调用f(int)
f(nullptr);  // 调用f(void*)
```

对于C++98来说，原则上不要在指针类型和整数之间重载，若NULL定义为0L，而从long到int、从long到bool，和从0L到void*都被视为同样好

`nullptr`的类型是`std::nullptr_t`，`std::nullptr_t`可以隐式转换到所有裸指针类型

```c++
typedef decltype(nullptr) nullptr_t;
```

在模板推导中，0和NULL都会被推导为整数，而不是指针

```c++
int f1(std::shared_ptr<Widget>);
double f2(std::unique_ptr<Widget>);
bool f3(Widget*);

f1(0);
f2(NULL);
f3(nullptr);

template<typename FuncType, typename PtrType>
decltype(auto) call(FuncType func, PtryType ptr)
{
  return func(ptr);
}

call(f1, 0);  // ERROR
call(f2, NULL);  // ERROR
call(f3, nullptr);
```


## 条款9：优先选用别名声明，而非typedef

```c++
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS;
using UPtrMapSS = std::unique<std::unordered_map<std::string, std::string>>;

typedef void (*FP)(int, const std::string&);
using FP = void (*)(int, const std::string&);
```

别名声明可以模板化（别名模板）

```c++
template<typename T>
struct AllocList {
  typedef std::list<T, Alloc<T>> type;
};
AllocList<Widget>::type lw;

template<typename T>
using AllocList = std::list<T, Alloc<T>>;
AllocList<Widget> lw;
```

C++规定带依赖的类型前面必须要加typename，而别名模板必然是类型，因此不需要加

```c++
template<typename T>
class Widget {
private:
  typename AllocList<T>::type list;  // typedef
};

template<typename T>
class Widget {
private:
  AllocList<T> list;  // using
}
```

C++11中type trait是用typedef实现，因此要加type。C++14中加入_t的using实现

```c++
std::remove_const<T>::type  // C++11
std::remove_const_t<T>  // C++14

template<typename T>
using remove_const_t = typename remove_const<T>::type;
```


## 条款10：优先选用限定作用域的枚举类型，而非不限作用域的枚举类型

一般情况下，名字的可见性被限定在大括号内，但枚举量的名字属于枚举类型的作用域

```c++
enum Color { black, white };  // black和Color作用域相同
enum class Color { black, white };  // black的作用域限制在Color内
```

enum的枚举量可以隐式转换为整数，但enum class是更强类型，不存在隐式转换，除非`static_cast`

默认情况下，enum class的默认底层类型是int，而enum没有默认，编译器通常会选择能够表示枚举取值范围的最小底层类型。在需要前置声明的情况下，enum class可以不显式指定底层类型，因为有默认，但enum必须声明，否则报错

```c++
enum Color;  // ERROR
enum class Color;

enum Color: std::uint8_t;
enum class Status: std::uint32_t;
```

当指定`std::tuple`的域时，enum是有优势的

```c++
enum UserInfoFields { uiName, uiEmail };
using UserInfo = std::tuple<std::string, std::string>;

UserInfo info;
auto val = std::get<uiEmail>(info);
```

```c++
enum class UserInfoFields { uiName, uiEmail };
UserInfo info;
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(info);

template<typename E>
constexpr auto toUType(E enumerator) noexcept
{
  return static_cast<std::underlying_type_t<E>>(enumerator);
}
auto val = std::get<toUType(UserInfoFields::uiEmail)>(info);
```


## 条款11：优先选用删除函数，而非private未定义函数

为了阻止用户调用复制构造或者复制赋值等函数，C++98可以把成员函数定义为private，并且只声明不定义

C++11更好的方式是使用delete，当调用成员函数时，C++会首先检查可访问性，后检查删除性，因此把delete放在public可以获得更好的错误信息

private只能用于成员函数，而delete可以用于所有函数，包括模板特化。要留意的是，即使是delete的方法，重载决议的时候也会使用

```c++
bool isLucky(int);
bool isLucky(char) = delete;
bool isLucky(double) = delete;
// 假如没有delete，可以传double隐式转换为int调用，加了delete后，重载决议会使用该版本并报错

template<typename T>
void process(T* ptr);

template<>
void process<void>(void*) = delete;

template<>
void process<char*>(char*) = delete;

template<>
void process<const char*>(const char*) = delete;
```

如果是成员模板函数，用private是达不到delete的效果，因为模板特化要求与主模板相同的名空间，而非类作用域

```c++
class Widget {
public:
  template<typename T>
  void process(T* ptr) {}

private:
  template<>
  void process<void>(void*);  // ERROR
};

class Widget {
public:
  template<typename T>
  void process(T* ptr) {}
};

template<>
void Widget::process<void>(void*) = delete;
```


## 条款12：为意在改写的函数添加override声明

override的条件
+ 基类的函数必须是虚函数（子类可写可不写）
+ 函数名必须相同（析构函数除外）
+ 函数形参类型完全相同
+ 函数const完全相同
+ 返回值和异常定义完全相同（基类noexcept，子类不能放宽，但子类可以增加noexcept）
+ C++11函数引用饰词完全相同

override和final是语境关键字，就是特定语境下才是保留关键字，因此可以定义名字为override的方法

final用于虚函数可以阻止在派生类被改写，修饰类的时候可以禁止继承

```c++
class Widget {
public:
  using DataType = std::vector<double>;

  DataType& data() &
  { return values; }  // 左值类型返回左值

  DataType data() &&
  { return std::move(values); }  // 右值类型返回右值

private:
  DataType values;
};
```


## 条款13：优先选用const_iterator，而非iterator

C++98对`const_iterator`不够全面，使用受限

```c++
// C++98不支持using
typedef std::vector<int>::iterator IterT;
typedef std::vector<int>::const_iterator ConstIterT;

std::vector<int> values;
ConstIterT ci = std::find(
  static_cast<ConstIterT>(values.begin()),  // C++98还没有cbegin
  static_cast<ConstIterT>(values.end()),
  1983);
values.insert(static_cast<IterT>(ci), 1998);
// 不支持const_iterator，同时编译也会出错，即使用reinterpret_cast也无法完成
```

C++11对`const_iterator`的支持

```c++
std::vector<int> values;
auto it = std::find(values.cbegin(), values.cend(), 1983);
values.insert(it, 1998);
```

C++14新增`std::cbegin`和`std::cend`（C++11只有`std::begin`和`std::end`）

```c++
template<typename C, typename V>
void findAndInsert(C& container, const V& targetVal, const V& insertVal)
{
  using std::cbegin;
  using std::cend;

  // 数组也适用
  auto it = std::find(cbegin(container), cend(container), targetVal);
  container.insert(it, insertVal);
}
```

`std::cbegin`的实现也简单，`std::begin`是有const的重载版本

```c++
template<typename _Container>
inline constexpr auto
cbegin(const _Container& __cont) noexcept(noexcept(std::begin(__cont)))
  -> decltype(std::begin(__cont))
  { return std::begin(__cont); }

// noexcept(expr)，若expr为true则不抛出异常
```


## 条款14：只要函数不会发射异常，就为其加上noexcept声明

noexcept和const一样，也属于接口设计的一部分，可能抛出异常的函数加上noexcept属于设计缺陷，noexcept状态会影响调用代码的异常安全性和运行效率

```c++
int f(int) throw();  // C++98
int f(int) noexcept;  // C++11
```

若异常从f抛出，则违反了f的异常规格定义。C++98下，调用栈会解开至f的调用方，而C++11则不会调用`std::unexpected`并且可能或可能不进行栈回溯。noexcept声明的函数，不需要异常传出函数前保持栈可以解开；也不用保证在异常抛出时，所有对象按构造顺序的逆序完成析构。

某些函数声明为noexcept会收益巨大。`push_back`提供强异常安全保证，当空间不够时，会分配新的内存，把老的数据复制到新的内存后，才会释放老的内存。C++11的移动操作可以避免复制的消耗，但把复制替换为移动，若移动操作抛出异常，则会违反异常安全的保证，所以标准容器采用的是能移动就移动（确保不抛出异常），不能移动就复制的策略。实际调用的是`std::move_if_noexcept`（通过`std::is_nothrow_move_constructible`判断移动构造是否指定noexcept），而不是`std::move`

```c++
std::vector<Widget> v;
Widget w;
v.push_back(w);
```

`swap`也取决于用户定义的`swap`是否带有noexcept

```c++
template<class T, size_t N>
void swap(T (&a)[N], T (&b)[N]) noexcept(noexcept(swap(*a, *b)));

template<class T1, class T2>
struct pair {
  void swap(pair& p) noexcept(
    noexcept(swap(first, p.first)) &&
    noexcept(swap(second, p.second))
  )
};
```

大部分函数是异常中立的，自身不抛出异常，但调用的函数可能抛出异常，不能强行加上noexcept

部分函数默认就是noexcept
+ 析构函数，除非基类或成员的析构函数为noexcept(false)
+ 默认构造函数、复制构造函数、移动构造函数，除非构造函数的调用的基类或成员的构造函数为潜在抛出；初始化的子表达式，例如默认参数表达式，为潜在抛出；默认成员初始化器为潜在抛出
+ 默认复制赋值运算符、移动赋值运算符，除非隐式定义中的任何赋值运算符的调用为潜在抛出
+ operator delete

即使声明了noexcept，也允许调用没有声明noexcept的函数，但抛出异常时会调用`std::terminate`（无法catch）


## 条款15：只要有可能使用constexpr，就使用它

constexpr对象都具有const属性，并且是编译期用已知的值实现初始化

const只表示不可修改，并不代表值在编译期已知，可能是运行期计算的

```c++
int sz;
constexpr auto size = sz;  // ERROR
std::array<int, sz> data;  // ERROR

constexpr auto size = 10;
std::array<int, size> data;

const auto size = sz;
std::array<int, size> data;  // ERROR
```

constexpr函数若实参都是编译期已知的，则结果是编译期计算出来的。如果一个或多个实参在编译期未知，则和普通函数没有区别。`std::array`可以用来验证是否真的是编译期计算

```c++
constexpr int pow(int base, int exp) noexcept;

constexpr auto num = 5;
std::array<int, pow(3, 5)> result;

auto base = read("base");
auto exp = read("exp");
auto value = pow(base, exp);  // 运行期
```

C++11限制`constexpr`不得多于一条执行语句，可以用`?`实现`if`，用递归实现循环。C++14放宽了限制

```c++
// C++11
constexpr int pow(int base, int exp) noexcept
{
  return exp == 0 ? 1 : base * pow(base, exp - 1);
}

// C++14
constexpr int pow(int base, int exp) noexcept
{
  auto result = 1;
  for (int n = 0; n < exp; ++n) result *= base;
  return result;
}
```

constexpr要在编译期完成要求传入和返回的都是字面量，C++内建的类型均符合（C++11中void不是，C++14解除限制），用户自定义类型也可以是字面量。C++11中constexpr都隐式声明为const，C++14解除限制。

```c++
class Point {
public:
  constexpr Point(double x, double y) noexcept
    : x_(x), y_(y) {}

  constexpr double x() const noexcept { return x_; }
  constexpr double y() const noexcept { return y_; }

  constexpr void setX(double x) noexcept { x_ = x; }  // C++14
  constexpr void setY(double y) noexcept { y_ = y; }  // C++14

private:
  double x_, y_;
};
```

```c++
constexpr Point p1(1.0, 2.0);
constexpr Point p2(3.0, 4.0);

constexpr Point mid(const Point& p1, const Point& p2)
{
  return { (p1.x() + p2.x()) / 2, (p1.y() + p2.y()) / 2 };
}

// 构造，访问器，普通函数调用都是在编译期完成
constexpr auto mid = mid(p1, p2);
```

```c++
constexpr Point reflection(const Point& p) noexcept
{
  Point result;

  result.setX(-p.x());
  result.setY(-p.y());

  return result;
}

constexpr auto reflected = reflection(mid);
```

constexpr属于函数签名的一部分，移除后可能导致原有的代码无法通过（在函数内加入I/O语句也可能导致constexpr失效）


## 条款16：保证const成员函数的线程安全性

`const`事实上承诺了线程安全性（除非保证不会在多个线程中使用），`mutable`可能会破坏了这个承诺，导致数据竞争。

```c++
class Polynomial
{
public:
  using RootsType = std::vector<double>;

  // 需要mutex或者atomic保护（单个变量）
  RootsType roots() const
  {
    if (!rootsAreValid)
    {
      // cache miss
      rootsAreValid = true;
    }
    return rootsVal;
  }

private:
  // 与存储数据无关，定义为mutable
  mutable bool rootsAreaValid { false };
  mutable RootsType rootsVal {};
};
```


## 条款17：理解特种成员函数的生成机制

特种成员函数是指C++会自行生成的成员函数。只有在需要的时候生成，即使用了但未显式声明。默认是public、inline且非虚（除非是析构函数，且基类也是虚函数）。

+ 默认构造函数
+ 析构函数：C++11默认为`noexcept`
+ 复制构造函数
+ 复制赋值函数
+ 移动构造函数：`T(T&& rhs)`
+ 移动赋值函数：`T& operator=(T&& rhs)`

移动操作并不能保证每个成员都是移动，如果不能移动的话会执行成员复制

复制构造函数和复制赋值函数是相互独立的，声明其中一个并不会阻止另外一个生成。如果声明了移动操作，会阻止复制操作的生成

移动构造函数和移动赋值函数并不独立，声明其中一个会阻止另一个生成

三大律（Rule of Three）：如果声明了复制构造函数、复制赋值函数，或析构函数的其中一个，则要同时声明所有三个函数。如果声明了析构函数，则应当阻止复制操作自动生成。C++98并没有执行该原则，C++11保持了兼容，但移动操作则被禁止生成

因此，移动操作仅当未声明任何复制操作、移动操作和析构函数，且需要时才会生成。

使用`= default`可以解除限制

```c++
class Base
{
public:
  virtual ~Base() = default;

  Base(Base&&) = default;
  Base& operator=(Base&&) = default;

  Base(const Base&) = default;
  Base& operator=(const Base&) = default;
};

```

同时较好的实践是，如果需要自动生成的地方，显式进行声明，并使用`= default`进行定义

```c++
class StringTable
{
public:
  StringTable() {}
  ~StringTable() {}
  // 迭代过程中增加的析构函数，会抑制移动操作的生成
  // 则原来自动生成的移动操作都会变成复制操作，造成性能下降

private:
  std::map<int, std::string> values;
};
```

成员函数模板在任何情况下都不会阻止特种成员函数的生成，即使模板实例化之后和复制构造函数签名完全相同

```c++
class Widget {
  template<typename T>
  Widget(const T& rhs);

  template<typename T>
  Widget& operator=(const T& rhs);
};
```


# 智能指针

## 条款18：使用std::unique_ptr管理具备专属所有权的资源

可以认为默认情况下`std::unique_ptr`和裸指针有相同的尺寸，并且大多数的操作都是精确执行了相同的指令。

不允许复制，只可以移动。默认通过`delete`析构，可以自定义析构函数，同时析构函数类型是指针类型的一部分。禁止裸指针和`std::unique_ptr`之间的隐式转换，需要使用`reset`。

```c++
template<typename... Ts>
auto makeInvestment(Ts&&... params)
{
  auto delInvmt = [](Investment *pInvestment)
  {
    makeLogEntry(pInvestment);
    delete pInvestment;
  };

  std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);
  if (...)
  {
    pInv.reset(new Stock(std::forward<Ts>(params)...));
  }
  else
  {
    pInv.reset(new RealEstate(std::forward<Ts>(params)...));
  }
  return pInv;
}
```

如果析构函数是函数指针，`std::unique_ptr`大小会增大一到两个word；如果是函数对象，则取决于存储的状态。对于无状态的函数对象，例如无捕获的lambda，不会增加任何存储空间，优先选择。

支持数组形式`std::unique_ptr<T[]>`，但应该优先选择容器，如`std::array`、`std::vector`等。

可以方便高效地转换为`std::shared_ptr`，适合工厂方法。

```c++
std::shared_ptr<Investment> sp = makeInvestment(arguments);
```


## 条款19：使用std::shared_ptr管理具备共享所有权的资源

对性能的影响
+ `std::shared_ptr`大小是裸指针的两倍（指向引用计数的指针）
+ 引用计数的内存需要动态分配，因为原对象并没有空间存储。使用`std::make_shared`可以避免动态分配的成本
+ 引用计数的操作是原子的

移动构造函数会比复制构造函数快，因为不需要递增引用计数。

默认使用`delete`析构，也支持自定义，同时析构函数类型并不是指针类型的一部分，因此不同析构函数的指针也可以相互复制，或者放在同一个容器内。

```c++
auto loggingDel = [](Widget *pw)
{
  makeLogEntry(pw);
  delete pw;
};

std::shared_ptr<Widget> spw(new Widget, loggingDel);  // 类型不包含析构函数
```

自定义析构函数不会改变`std::shared_ptr`的大小，因为这部分数据存储在位于堆上的控制块。

控制块的创建时机
+ `std::make_shared`总是创建
+ 从`std::unique_ptr`构造
+ 从裸指针构造

从裸指针可以构造出多个`std::shared_ptr`，造成多个控制块，未定义行为。

```c++
auto pw = new Widget;

std::shared_ptr<Widget> spw1(pw, loggingDel);
std::shared_ptr<Widget> spw2(pw, loggingDel);  // 另一个引用计数
```

可以使用`std::make_shared`避免（自定义析构的时候无法使用），或者使用`new`的结果

```c++
std::shared_ptr<Widget> spw1(new Widget, loggingDel);
std::shared_ptr<Widget> spw2(spw1);
```

从`this`指针构造`std::shared_ptr`可能会造成未定义行为。

```c++
std::vector<std::shared_ptr<Widget>> processedWidget;

class Widget
{
public:
  void process()
  {
    processedWidget.emplace_back(this);  // ERROR
  }
};
```

使用`std::enable_shared_from_this`，`shared_from_this`只有在包含控制块的时候调用才是合法，否则是未定义行为或者抛出`std::bad_weak_ptr`

为了确保构造完成后必定绑定了`std::shared_ptr`，把构造函数定义为private

```c++
class Widget: public std::enbale_shared_from_this<Widget>
{
public:
  template<typename... Ts>
  static std::shared_ptr<Widget> create(Ts&&... params);

  void process()
  {
    processWidget.emplace_back(shared_from_this());
  }

private:
  // constructor
}
```

因为`std::shared_from_this`的实现是包含了一个`std::weak_ptr`，`std::weak_ptr`的初始化依赖于`std::shared_ptr`，成员`std::weak_ptr`是在第一个`std::shared_ptr`构造的时候赋值

```c++
template<typename _Yp>
using __esft_base_t = decltype(__enable_shared_from_this_base(
  std::declval<const __shared_count<_Lp>&>(),
  std::declval<_Yp*>()));

// Detect an accessible and unambiguous enable_shared_from_this base.
template<typename _Yp, typename = void>
struct __has_esft_base
  : false_type { };

template<typename _Yp>
struct __has_esft_base<_Yp, __void_t<__esft_base_t<_Yp>>>
  : __not_<is_array<_Tp>> { }; // No enable shared_from_this for arrays

template<typename _Yp, typename _Yp2 = typename remove_cv<_Yp>::type>
typename enable_if<__has_esft_base<_Yp2>::value>::type
_M_enable_shared_from_this_with(_Yp* __p) noexcept
{
  if (auto __base = __enable_shared_from_this_base(_M_refcount, __p))
    __base->_M_weak_assign(const_cast<_Yp2*>(__p), _M_refcount);
}

template<typename _Yp, typename _Yp2 = typename remove_cv<_Yp>::type>
typename enable_if<!__has_esft_base<_Yp2>::value>::type
_M_enable_shared_from_this_with(_Yp*) noexcept
{ }
```


## 条款20：对于类似std::shared_ptr但有可能空悬的指针使用std::weak_ptr

`std::weak_ptr`通过`std::shared_ptr`创建，通过`expired`判定是否失效（但判定和dereference之间依然存在竞争，需要原子操作完成）

```c++
auto spw = std::make_shared<Widget>();
std::weak_ptr<Widget> wpw(spw);

auto spw1 = wpw.lock();
std::shared_ptr<Widget> spw2(wpw);
```

用`std::weak_ptr`构造缓存

```c++
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
  static std::unordered_map<WidgetID, std::weak_ptr<const Widget>> cache;
  auto objPtr = cache[id].lock();
  if (!objPtr)
  {
    objPtr = loadWidget(id);
    cache[id] = objPtr;
  }
  return objPtr;
}
```

可以解除循环引用，但如果能严格控制生存周期的话则不需要。例如，树结构的父子节点，若子节点只被父节点引用，则父节点可持有子节点的`std::unique_ptr`，子节点持有父节点的裸指针。


## 条款21：优先使用std::make_unique和std::make_shared，而非直接使用new

`std::allocate_shared`行为与`std::make_shared`一样，但第一个实参是内存分配器。

`std::make_shared`可以确保异常安全

```c++
processWidget(std::shared_ptr<Widget>(new Widget), computePriority());
```

编译器执行的顺序有可能是
1. `new Widget`
2. `computePriority`
3. `std::shared_ptr`构造

若2出现异常，则1泄露。

```c++
// 保证异常安全的实现
std::shared_ptr<Widget> spw(new Widget, cusDel);
processWidget(std::move(spw), computePriority());
```

`std::make_shared`有利于提高性能，`std::shared_ptr<Widget> spw(new Widget)`是两次内存分配（包含一次控制块分配），`std::make_shared`则可以一次分配完成。

如果需要自定义析构函数或者要使用`std:;initializer_list`，则不能使用`std::make_shared`

```c++
// 变通方案
auto initList = {10, 20};
auto spv = std::make_shared<std::vector<int>>(initList);
```

其他不适用的场景
1. 自定义了`operator new`和`operator delete`的类
2. 如果使用了`std::weak_ptr`，则内存会延迟回收（因为控制块和对象是一次内存分配）


## 条款22：使用Pimpl习惯用法时，将特殊成员函数的定义放到实现文件中

```c++
// header
class Widget
{
public:
  Widget();
  ~Widget();

  Widget(const Widget& rhs);
  Widget& operator=(const Widget& rhs);

  Widget(Widget&& rhs);
  Widget& operator=(Widget&& rhs);

private:
  struct Impl;
  std::unique_ptr<Impl> pImpl;
};

// source
#include <string>
#include <vector>

struct Widget::Impl
{
  std::string name;
  std::vector<double> data;
};

Widget::Widget
  : pImpl(std::make_unique<Impl>())
{}

Widget::Widget(const Widget& rhs)
  : pImpl(std::make_unique<Impl>(*rhs.pImpl))
{}

Widget& Widget::operator=(const Widget& rhs)
{
  *pImpl = *rhs.pImpl;
  reutrn *this;
}

Widget::~Widget() = default;
Widget::Widget(Widget&& rhs) = default;
Widget& Widget::operator=(Widget&& rhs) = default;
```

几个特别的地方
1. 左值的复制和赋值函数不能用default，因为`std::unique_ptr`不可复制
2. `std::unique_ptr`支持不完整的类型，这里不在头文件实现析构函数和右值复制构造/赋值函数，是因为合成的析构函数会调用`std::unique_ptr`的析构，对象的析构函数是`std::unique_ptr`的类型一部分（为了产生更快的代码），若在头文件中实现，则无法观察到对象的实现，中`static_assert`。而使用`std::shared_ptr`则不会有这个问题，因为析构函数不是类型的一部分。


# 右值引用、移动语义和完美转发

形参总是左值，即使类型是右值引用

```c++
void f(Widget&& w);
```

## 条款23：理解std::move和std::forward

`std::move`和`std::forward`不会生成任何可执行代码，仅执行强制类型转换

`std::move`就是把实参无条件强制转为右值，返回值要先`remove_reference`是因为`&&`是万能引用，如果传入的是左值引用就会导致返回的是左值引用

```c++
template<typename T>
decltype(auto) move(T&& param)
{
  using ReturnType = remove_reference_t<T>&&;
  return static_cast<ReturnType>(param);
}
```

即使使用了右值，也并不代表是不进行复制的。

```c++
class Annotation(const std::string text)
{
public:
  explicit Annotation(const std::string text)
    : value(std::move(text)) {}

private:
  std::string value;
};

class string
{
public:
  string(const string& rhs);
  string(string&& rhs);
};

// move之后仍然是const的右值，只能选择常量左值引用的复制构造函数，仍然是复制
// 如果需要某个对象是需要移动的话，最好不要声明为const，否则可能会变成复制操作
```

`std::forward`是有条件转换，在实参是用右值完成初始化时进行强制类型转换。`std::forward`是根据模板参数`T`判断传入的是左值还是右值。

```c++
void process(const Widget& lvalArg);
void process(Widget&& rvalArg);

template<typename T>
void logAndProcess(T&& params)
{
  // ...
  // params是左值，如果是用右值初始化时需要保持右值转发
  process(std::forward<T>(param));
}
```

理论上可以用`std::forward`实现`std::move`的所有功能

```c++
class Widget {
public:
  Widget(Widget&& rhs)
    : s(std::move(rhs.s)) {}

private:
  std::string s;
};

// 理论上等价
class Widget {
public:
  Widget(Widget&& rhs)
    : s(std::forward<std::string>(rhs.s)) {}

private:
  std::string s;
};
```

`std::move`是包含类型推导，是无条件转换为右值，`std::forward`是根据模板参数进行转换，可能为左值也可能为右值


## 条款24：区分万能引用和右值引用

万能引用（forwarding reference）出现在需要类型推导的地方

```c++
// 右值引用
void f(Widget&& param);
Widget&& var1 = Widget();
template<typename T>
void f(std::vector<T>&& params);

// 万能引用
auto&& var2 = var1;
template<typename T>
void f(T&& param);
```

左值对应左值引用，右值对应右值引用

```c++
template<typename T>
void f(T&& param);

Widget w;
f(w);  // param类型是Widget&，T是Widget&
f(std::move(w));  // param类型是Widget&&，T是Widget
```

`T&&`不一定就是万能引用，必须是涉及类型推导

```c++
template<class T>
class A {
  template<class U>
  A(T&& x, U&& y);  // x是右值引用，y是万能引用
};
```

`auto&&`在推导初始化列表的情况下不是万能引用

```c++
auto&& z = {1, 2, 3};  // 不是万能引用

for (auto&& x: f()) {
  // x是万能引用，使用范围for循环的最安全方式
}

auto invocation =
  [](auto&& func, auto&&... params)
{
  std::forward<decltype(func)>(func)(
    std::forward<decltype(params)>(params)...
  );
};
```


## 条款25：针对右值引用实施std::move，针对万能引用实施std::forward

右值引用的形参表明对象可移动，传递的时候通过`std::move`无条件转换为右值

```c++
class Widget {
public:
  Widget(Widget&& rhs)
  : name(std::move(rhs.name)), p(std::move(rhs.p))
  {}

private:
  std::string name;
  std::shared_ptr<Data> p;
};
```

`std::forward`转发万能形参，有条件地转换为右值

```c++
class Widget {
public:
  template<typename T>
  void setName(T&& value)
  { name = std::forward<T>(value); }
  // { name = std::move(value); }  ERROR，把形参强制转为右值，但实参可能为左值
};

class Widget {
public:
  void setName(const std::string& value)
  { name = value; }

  void setName(std::string&& value)
  { name = std::move(value); }
};
// 相比万能引用版本，代码量变大
// 同时对于w.setName("abc")调用更慢，多了一个std::string临时对象的构造和析构
```

如果引用要被多次使用的话，要保证最后才被移走

```c++
template<typename T>
void setText(T&& text)
{
  sign.setText(text);  // 不移走
  signHistory.add(std::forward<T>(text));  // 有条件转换
}
```

如果返回的是绑定到右值引用或者万能引用的对象，则返回前应该调用`std::move`或者`std::forward`，尽可能使用移动，即使不能移动也会调用复制，不会出错

```c++
Matrix operator+(Matrix&& lhs, const Matrix& rhs)
{
  lhs += rhs;
  return std::move(lhs);  // 移动构造
}

template<typename T>
Fraction reduceAndCopy(T&& frac)
{
  frac.reduce();
  return std::forward<T>(frac);  // 右值移动，左值复制
}
```

局部变量的返回已经有RVO，不需要调用`std::move`，如果用了反而会阻止RVO优化。因为RVO要求对象类象和返回值完全相同，`std::move`返回的是引用类型，导致不必要的复制或者移动。标准规定，如果不能省略复制时（例如多个执行路径返回不同的对象），返回的对象必须作为右值处理，因此这种情况下可以直接按值返回，同样不需要`std::move`。


## 条款26：避免依万能引用类型进行重载

```c++
std::multiset<std::string> names;

void add(const std::string& name)
{
  names.emplace(name);
}

std::string name("abc");
add(name);  // 左值复制
add(std::string("xyz"));  // 右值复制
add("zxc");  // 构造临时对象加复制
```

使用万能引用提高效率

```c++
template<typename T>
void add(T&& name)
{
  names.emplace(std::forward<T>(name));
}

std::string name("abc");
add(name);  // 左值复制
add(std::string("xyz"));  // 右值移动
add("zxc");  // 直接构造，没有临时对象
```

形参为万能引用的函数，是匹配中最贪婪的，几乎会和任何实参类型产生精确匹配

```c++
template<typename T>
void add(T&& name);
void add(int idx);

add(1);  // int重载版本

short x = 1;
add(x);  // ERROR: T&&版本
// T&&可生成short&的精确匹配版本，而int需要进行类型提升
```

```c++
class Person {
public:
  template<typename T>
  explicit Persion(T&& n)
    : name(std::forward<T>(n)) {}

  // Person(const Persion& rhs); 模板构造函数并不会阻止编译器生成复制构造函数
  // Person(Persion&& rhs);
};

Person p("abc");
auto copy(p);  // ERROR: 调用的是完美转发函数Persion&，而复制构造函数需要添加const才能匹配

const Persion p("abc);
auto copy(p);  // 复制构造函数
```

```c++
class SpecialPerson: public Persion {
public:
  // 调用的均是基类的完美转发构造函数
  SpecialPerson(const SpecialPerson& rhs)
    : Person(rhs) {}  
  SpecialPerson(SpecialPerson&& rhs)
    : Person(std::move(rhs)) {}
};
```

尽量避免把万能引用作为重载函数的形参选择


## 条款27：熟悉按万能引用类型进行重载的替代方案

+ 不重载的就没有问题，可以使用不同的函数名，但构造函数无法解决

+ 使用const T&类型的形参，但性能不如万能引用

+ 如果肯定要复制形参的话，使用按值传递

+ 使用标签分派，事实上就是对外的接口不使用重载，在内部实现上增加参数区分要调用的版本

```c++
template<typename T>
void add(T&& name)
{
  addImpl(std::forward<T>(name), std::is_integral<std::remove_reference_t<T>>());
}

template<typename T>
void addImpl(T&& name, std::false_type)
{
  names.emplace(std::forward<T>(name));
}

std::string nameFromIdx(int idx);
template<typename T>
void addImpl(int idx, std::true_type)
{
  add(nameFromIdx(idx));  // 委托回原函数
}
```

+ 使用SFINAE，对模板匹配进行限制

```c++
// decay_t
// 若T指名U的数组或到U的数组的引用类型，则成员typedef type 为 U*
// 否则，若T为函数类型F或到它的引用，则成员typedef type为std::add_pointer<F>::type
// 否则，成员typedef type为 std::remove_cv<std::remove_reference<T>::type>::type

class Person {
public:
  // 避免覆盖复制构造函数
  template<
    typename T,
    std::enable_if_t<!std::is_same<Person, std::decay_t<T>>::value>
  >
  explicit Person(T&& n);

  // 考虑继承的情况
  template<
    typename T,
    std::enable_if_t<!std::is_base_of<Person, std::decay_t<T>>::value>
  >
  explicit Person(T&& n);

  // 限制其他参数情况
  template<
    typename T,
    std::enable_if_t<
      !std::is_base_of<Person, std::decay_t<T>>::value &&
      !std::is_integral<std::remove_reference_t<T>::value>
    >
  >
  explicit Person(T&& n)
    : name(std::forward<T>(n))
  {
    // 通过static_assert优化完美转发的错误信息
    static_assert(
      std::is_constructible<std::string, T>::value,
      "Parameter n cannot be used to consturct a std::string");
  }
};
```

标签指定和模板限制都利用了完美转发，没有损失效率，但可能存在不能转发的问题，出错时错误信息也会很难辨认


## 条款28：理解引用折叠

C++中，声明引用的引用是非法的，但当编译器有必要的时候，会进行引用折叠，变成单个引用

引用折叠可能出现在四种语境
+ 模板实例化
+ auto类型推导
+ typedef和别名声明
+ decltype

引用折叠的规则是，原始引用中有任一为左值引用，结果为左值引用，否则为右值引用

```c++
template<typename T>
void func(T&& param);
```

如果传递的实参是左值，则`T`为左值引用；若实参是右值，则`T`是非引用类型（其他语境完全相同，例如`auto&&`，`auto`可以看做是`T`）

```c++
/**
 *  @brief  Forward an lvalue.
 *  @return The parameter cast to the specified type.
 *
 *  This function is used to implement "perfect forwarding".
 */
template<typename _Tp>
  constexpr _Tp&&
  forward(typename std::remove_reference<_Tp>::type& __t) noexcept
  { return static_cast<_Tp&&>(__t); }
 /**
 *  @brief  Forward an rvalue.
 *  @return The parameter cast to the specified type.
 *
 *  This function is used to implement "perfect forwarding".
 */
template<typename _Tp>
  constexpr _Tp&&
  forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
  {
    static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
          " substituting _Tp is an lvalue reference type");
    return static_cast<_Tp&&>(__t);
  }

/**
 *  @brief  Convert a value to an rvalue.
 *  @param  __t  A thing of arbitrary type.
 *  @return The parameter cast to an rvalue-reference to allow moving it.
*/
template<typename _Tp>
  constexpr typename std::remove_reference<_Tp>::type&&
  move(_Tp&& __t) noexcept
  { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }
```

`std::forward`的实现，`static_assert`是为了禁止右值引用转换为左值引用。`std::forward`的本质就是根据模板参数`T`加上`&&`进行引用折叠（`T`要指定），而`std::move`则是无条件转为右值（`T`根据实参推导）。


## 条款29：假定移动操作不存在、成本高、未使用

在几个场景下，C++11的移动操作不会带来优化

+ 对象没有提供移动操作，移动变成复制
+ 移动操作的实现并不是都比复制快，例如`std::vector`包含指定堆的指针，移动仅仅是指针复制，确实更快，而`std::array`则都是栈上的内容，无论移动还是复制都需要线性时间；`std::string`如果使用了SSO，移动同样是需要线性时间
+ 要求强异常安全的代码，由于移动没有加上noexcept，导致使用的是复制

在类型不确定的情况下，如模板编写，都应该假定使用的是复制，而不是移动，除非操作的类型明确并已知移动操作的支持情况


## 条款30：熟悉完美转发的失败情形

```c++
template<typename... Ts>
void fwd(Ts&&... params)
{
  f(std::forward<Ts>(params)...);
}
```

大括号初始化也会转发失败，因为形参没有声明为`std::initializer_list`或者准确的类型，编译器就拒绝从大括号表达式进行类型推导

```c++
void f(const std::vector<int>& v);

f({ 1, 2, 3 });
fwd({ 1, 2, 3 });  // ERROR

auto x = { 1, 2, 3 };  // 先进行类型推导
fwd(x);
```

0和NULL都会被推导为整数，因此都不能作为空指针进行完美转发，应使用`nullptr`

仅有声明的整型`static const`成员变量也会转发失败。`static const`只声明也能使用是因为编译器在需要的地方进行了替换。但假如出现了取址操作，则必须要定义，否则会链接失败。完美转发用的是万能引用，引用可以看做是解引用的指针，因此需要定义的存在。

```c++
class Widget {
public:
  static const std::size_t MinVals = 128;
};

f(Widget::MinVals);
fwd(Widget::MinVals);  // ERROR，用到了引用
```

转发重载的函数名字和模板名字也会失败

```c++
int process(int);
int process(int, int);

void f(int pf(int));

f(process);  // 编译期根据f的声明选择合适的函数指针
fwd(process);  // ERROR，没有类型推导，不能确定process

template<typename T>
T work(T param) {}

fwd(work);  // ERROR，不能确定work的实例

// 除非进行强制指定
using ProcessFuncType = int (*)(int);
ProcessFuncType processPtr = process;  // 指定了签名

fwd(processPtr);
fwd(static_cast<ProcessFuncType>(work));  // 指定了版本
```

位域作为函数实参完美转发会失败，因为标准规定非`const`引用禁止绑定位域。在硬件层面，引用和指针是同一事物。位域可能不对齐，不可能有创建指向任意bit的指针。要顺利转发必须是先构造位域的副本。

```c++
// 若位域的指定大小大于其类型的大小，则值为类型所限制，`std::uint8_t b : 1000`; 仍会保有 0 与 255 间的值。额外位成为不使用的填充位。

// 因为位域不必始于一个字节的开始，故不能取位域的地址。指向位域的指针和非`const`引用是不可行的。从位域初始化`const`引用时，临时量被创建（其类型是位域的类型），并以位域的值复制初始化，而引用绑定到该临时量。

// 位域的类型只能是整数或枚举类型；不能是静态数据成员；没有位域纯右值：左值到右值转换始终生成位域底层类型的对象。

// 以下属性是跟具体实现相关的
// + 以范围外的值赋值或初始化有符号位域，或从自增有符号位域越过其范围产生的值。
// + 任何关于类对象中位域的实际分配细节。例如，在某些平台上，位域不跨过字节，其他平台上会跨过；还有，在某些平台上，位域从左到右打包，其他为从右到左

struct S {
  // 将通常占用 2 字节：
  // 3 位： b1 的值
  // 2 位：不使用
  // 6 位： b2 的值
  // 2 位： b3 的值
  // 3 位：不使用
  unsigned char b1 : 3, : 2, b2 : 6, b3 : 2;
};

struct S {
  // 将通常占用 2 字节：
  // 3 位： b1 的值
  // 5 位：不使用
  // 6 位： b2 的值
  // 2 位： b3 的值
  unsigned char b1 : 3;
  unsigned char :0; // 开始新字节
  unsigned char b2 : 6;
  unsigned char b3 : 2;
};
```

# lambda表达式

+ lambda式：编译期
+ 闭包类：编译期
+ 闭包：运行期

```c++
// 同一lambda产生的闭包的副本，闭包进行复制
Type a;
auto x = [a] {};
auto y = x;
```

## 条款31：避免默认捕获模式

引用捕获了局部变量导致野指针

```c++
// &：以引用隐式捕获被使用的自动变量
// =：以复制捕获被使用的自动变量

using FilterContainer = std::vector<std::function<bool(int)>>;
FilterContainer filters;

void addFilter()
{
  auto divisor = computeDivisor();
  filters.emplace_back([&](int value) { return value % divisor == 0; })
  // &divisor能够更明显发现问题
}
```

如果引用的局部变量立刻被使用且不会被复制则没问题

```c++
template<typename C>
void processContainer(const C& container)
{
  auto divisor = computeDivisor();
  if (std::all_of(begin(container), end(container),
    [&](const auto& value) { return value % divisor == 0; }))  // 按值捕获也可以
  {
    // ...
  }
}
```

按值捕获并不一定就能避免野指针。捕获仅能针对创建lambda表达式作用域内可见的非静态局部变量。

```c++
class Widget
{
public:
  void addFilter() const
  {
    filters.emplace_back(
      [=](int value) { return value % divisor == 0; });
    // 捕获的实际上是this，如果显式写divisor或者&divisor均为编译错，依然有野指针的风险
    // 使用generalized lambda capture解决，[divisor = divisor](int value) {}
  }

private:
  int divisor;
}
```

使用默认捕获可能会误以为lamdba是独立的，但事实上可能涉及了static变量

```c++
void addFilter()
{
  static auto divisor = computeDivisor();
  filters.emplace_back(
    [=](int value) { return value % divisor == 0; }
  );
  ++divisor;
  // 看上去是按值捕获了divisor，但事实上并不能捕获
}
```


## 条款32：使用初始化捕获将对象移入闭包

C++11不支持移动捕获，C++14也不支持，但是可以用初始化捕获实现（即广义捕获，generalized lambda capture）

```c++
auto pw = std::make_unqiue<Widget>();
auto func = [pw = std::move(pw)]
  { returun pw->isValidated(); }
```

初始化捕获的`=`左右侧位于不同的作用域，左侧是闭包类，右侧是lambda定义处相同的作用域

C++11同样可以用仿函数实现，但代码量比较大。另一种模拟方式是用`std::bind`

```c++
std::vector<double> data;

// C++14
auto func = [data = std::move(data)] {}

// C++11
auto func = std::bind([](const std::vector<double>& data) {}, std::move(data));
```

默认lambda生成的闭包类的`operator()`是const，不能修改以复制捕获的参数，以及调用其非const的成员函数，可以加上`mutable`修饰

```c++
auto func = [x, v = std::vector<int>{}] {
  v.emplace_back(x++);  // ERROR const v & const x
};
```


## 条款33：对auto&&类型的形参使用decltype，以std::forward之

完美转发的lambda式

```c++
auto f =
  [](auto&& param)
  {
    return func(normalize(std::foward<decltype(params)>(params)...));
  };

// 等价的闭包类
class Generated {
public:
  template<typename T>
  auto operator()(T&&... params) const
  { return func(normalize(std::foward<decltype(params)>(params)...)); }
};
```

当传入左值时，decltype产生左值引用；当传入右值时，decltype产生右值引用

`std::forward`实质是对实参`statc_cast`为T&&，根据引用折叠规则可以确保左右值都正常转发


## 条款34：优先选用lambda式，而非std::bind

lambda的可读性更高

```c++
using Time = std::chrono::steady_clock::time_point;
enum class Sound { Beep, Siren, Whistle };
using Duration = std::chrono::steady_clock::duration;
void setAlarm(Time t, Sound s, Duration d);
```

lambda版本

```c++
auto setSoundL =
  [](Sound s)
  {
    using namespace std::chrono;
    using namespace std::literals;  // User-defined literals
    // 被处理成函数调用operator "" X(val)

    setAlarm(steady_clock::now() + 1h, s, 30s);
  }
```

`std::bind`版本

```c++
using namespace std::chrono;
using namespace std::literals;

auto setSoundB = std::bind(setAlarm, steady_clock::now() + 1h, _1, 30s);  // 逻辑错误
auto setSoundB = std::bind(
  setAlarm,
  std::bind(std::plus<>(), steady_clock::now(), 1h),  // C++14标准运算符模板实参大部分时候可以省略
  _1,
  30s);
```

如果包含重载版本，lamdba可以正确选择，`std::bind`则必须手工指定

```c++
using SetAlarmType = void(*)(Time t, Sound s, Duration d);

auto setSoundB = std::bind(
  static_cast<SetAlarmType>(setAlarm),
  std::bind(std::plus<>(), steady_clock::now(), 1h),  // C++14标准运算符模板实参大部分时候可以省略
  _1,
  30s);
```

lambda使用的是常规函数调用，而`std::bind`是通过函数指针，会被编译器内联的概率会更低，因此lambda生成的代码会更快。

```c++
auto betweenL =
  [lowVal, highVal](const auto& val)
  { return lowVal <= val && val <= highVal; };

using namespace std::placeholders;

auto betweenB = std::bind(
  std::logical_and<>(),
  std::bind(std::less_equal<>(), lowVal, _1),
  std::bind(std::less_equal<>(), _1, lowVal),
);
```

`std::bind`的参数总是默认复制，可以通过`std::ref`按引用存储，而lambda则是在声明时已经显式声明。

C++11中，`std::bind`合理使用的情况有
+ 移动捕获（C++14中可以用generalized capture）
+ 多态函数对象

```c++
class Widget {
public:
  template<typename T>
  void operator()(const T& param);
};

Widget w;
auto boundW = std::bind(w, _1);

boundW(1930);
boundW(nullptr);

// C++14中可以用auto解决
auto boundW =
  [w](const auto& param)
  { w(param); }
```


# 并发API

## 条款35：优先选用基于任务而非基于线程的程序设计


## 条款40：对并发使用std::atomic，对特种内存使用volatile

volatile不保证原子性

```c++
std::atomic<int> a(0);
a = 10;
std::cout << a;  // 读取是原子的，但整行代码并不是原子
++a;

// 均非原子操作，多线程未定义
volidata int b(0);
b = 10;
std::cout << b;
++b;
```

volatile不保证顺序性（编译器重排、CPU重排），atomic使用默认的顺序一致性模型则有保证

```c++
volatile bool valAvailable(false);
auto imptValue = computeValue();
valAvailable = true;  // 可能重排至imptValue赋值之前
```

常规内存
+ 如果向某个地址写入了值，值就会一直保持，直到被覆盖为止
+ 如果向某个地址写入了值，但期间没有读取过该地址，然后再次写入该地址，则第一次写入可以消除

编译器会针对常规内存的特征进行优化，而volatile则是通知编译器不要在该地址上进行优化

`std::atomic`的复制操作和复制赋值都被删除了（因此也没有移动操作）。因为读取是原子，赋值也是原子，因此整行代码应该也是原子，但硬件无法支持不同地址间的原子移动。

```c++
std::atomic<int> x;
auto y = x;  // ERROR

std::atomic<int> y(x.load());
```

volatile和`std::atomic`可以同时使用


# 微调

## 条款41：针对可复制的形参，在移动成本低并且一定会被复制的前提下，考虑将其按值传递

用于复制的形参，为了效率，提供左值和右值的重载版本，但需要维护两份实现

```c++
class Widget {
public:
  void addName(const std::string& name)
  { names.push_back(name); }

  void addName(std::string&& name)
  { names.push_back(std::move(name)); }

private:
  std::vector<std::string> names;
};
```

用万能引用方式，可以只维护一份实现，但因为使用了模板，要求在头文件实现，并且可能生成一些不需要的版本

```c++
class Widget {
public:
  template<typename T>
  void addName(T&& name)
  { names.emplace_back(std::forward<T>(name)); }
};
```

按值传递的方式

```c++
class Widget {
public:
  void addName(std::string name)
  { names.push_back(std::move(name)); }
};
```

效率对比，重载版本左值是一次复制，右值是一次移动；万能引用当参数为`std::string`时效率与重载版本一致，其他参数有原地构造的优势。按值传递比重载版本多了一次移动。

选择按值传递的前提在于
+ 形参是可复制的，当不可复制的时候就不需要维护两个版本，只实现可移版本即可

```c++
class Widget {
public:
  void setPtr(std::unique_ptr<std::string>&& ptr)
  { p = std::move(ptr); }  // 使用复制增加一次移动成本，没有必要
};
```

+ 传入的参数必定会被复制，无论参数最后是否被复制，按值传递方式在传参的时候就已经被复制了，如果代码存在分支，没有复制参数，则这次复制是浪费的，而引用传递则不会有这个消耗

+ 当移动成本低的时候才考虑，例如`std::string`开启SSO时，移动是线性消耗，而构造和赋值的消耗也可能有区别

```c++
class Password {
public:
  void changeTo(const std::string& pwd)
  { text = pwd; }
  // 如果text比pwd长度大时，不需要重新分配内存，而使用复制方式必定是要分配
};
```

+ 如果是接受基类类型及其派生类的话，不应该用按值传递，否则可能遇到切片问题



## 条款42：考虑置入而非插入

根本区别是，`push`接受的是待插入的对象，而`emplace`接受的是待插入的对象的构造函数实参，从而避免临时对象的创建和析构。

在需要构造的情况下，`push_back`消耗更高，首先要调用`std::string(const char *)`，然后调用`std::string(std::string&&)`，最后还需要调用析构函数。右值构造和析构造成了浪费。`emplace_back`表示的是原地构造，参数是通过完美转发用placement new进行构造，去除了临时对象的消耗。

```c++
std::vector<std::string> v;
v.push_back("abc");
```

`emplace`的优化在于去除了临时对象的创建和析构，但某些情况下，这个创建不可避免，因此也不会比`push`更快

+ 构造的值是通过赋值加入容器的

```c++
std::vector<std::string> v;
v.emplace_back();
v.emplace(v.begin(), "abc");
// 一般实现都不会选择在已有内存构造，而是通过移动赋值的方式加入容器，因此要构造作为待移动的对象 
```

+ 传递的实参和容器的内容物类型相同，此时`push`调用的就是`emplace`
+ 容器可能拒绝插入的值。例如要检查值是否重复，就要先构造临时对象进行查找

对于要求异常安全的操作，不建议使用`emplace`

```c++
std::vector<std::shared_ptr<Widget>> ptrs;

void killWidget(Widget *ptr);
ptrs.push_back(std::shared_ptr<Widget>(new Widget, killWidget));
ptrs.push_back({ new Widget, killWidget});
```

两种写法都能保证在容器处理之前裸指针已经被`std::shared_ptr`管理，即使插入过程中出现异常，也不会造成内存泄露
```c++
// 如果不需要自定义析构，则不需要显式new，就不会存在问题
ptrs.emplace_back(new Widget, killWidget);
```

而使用`emplace`在`new`和`std::shared_ptr`构造之间存在容器的相关代码，如果出现异常，则内存没有被正确回收

资源管理的原则是，在资源获取，和资源移交到资源管理对象之间，不能存在多余的代码
```c++
std::share_ptr<Widget> spw(new Widget, killWidget);
ptrs.push_back(std::move(spw));  // emplace_back也可以
```

`emplace`是可以调用`explicit`构造函数的，因为本质上`emplace`就是使用传递的实参进行直接初始化；而`push`调用的是复制初始化，会被阻止调用`explicit`。某些时候这种特性可能会造成隐藏的问题。

```c++
explicit regex(const char *);

std::regex r1 = nullptr;  // ERROR
std::regex r2(nullptr);

std::vector<std::regex> regexes;
regexes.push_back(nullptr);  // ERROR
regexes.emplace_back(nullptr);
```
