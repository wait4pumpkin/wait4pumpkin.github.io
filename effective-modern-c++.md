## 类型推导

### 条款1：理解模板类型推导

```c++
template<typename T>
void f(ParamType param);  // ParamType可能带有修饰词

f(expr);  // 从expr推导T和ParamType类型
```

#### 情况1：ParamType是指针或引用，但不是万能引用

### 条款4：掌握查看类型推导结果的方法

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


## auto

### 条款5：优先选用auto，而非显示类型声明

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


### 条款6：当auto推导的类型不符合要求时，使用带显式类型的初始化物习惯用法

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


## 转向现代C++

### 条款7：在创建对象时注意区分()和{}

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


### 条款8：优先选用nullptr，而非0或NULL

0和NULL都不是指针类型，只有C++发现在只能使用指针的环境发现0或者NULL，才会解释为指针。

xxx


### 条款16：保证const成员函数的线程安全性

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


### 条款17：理解特种成员函数的生成机制

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



## 智能指针

### 条款18：使用std::unique_ptr管理具备专属所有权的资源

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


### 条款19：使用std::shared_ptr管理具备共享所有权的资源

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


### 条款20：对于类似std::shared_ptr但有可能空悬的指针使用std::weak_ptr

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


### 条款21：优先使用std::make_unique和std::make_shared，而非直接使用new

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


### 条款22：使用Pimpl习惯用法时，将特殊成员函数的定义放到实现文件中

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


## 右值引用、移动语义和完美转发

形参总是左值，即使类型是右值引用

```c++
void f(Widget&& w);
```

### 条款23：理解std::move和std::forward

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

XXXXX P155 为什么forward<std::string>


### 条款24：区分万能引用和右值引用

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


## lambda表达式

+ lambda式：编译期
+ 闭包类：编译期
+ 闭包：运行期

### 条款31：避免默认捕获模式


## 并发API

### 条款35：优先选用基于任务而非基于线程的程序设计



## 微调

<!-- ### 条款41：针对可复制的形参，在移动成本低并且一定会被复制的前提下，考虑将其按值传递 -->


条款42：考虑置入而非插入

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
