# make_visitor实现

```c++
std::variant<int, std::string> v;

struct Visitor
{
    void operator()(int)
    {
        // ...
    }

    void operator()(std::string)
    {
        // ...
    }
};
std::visit(Visitor{}, v);
```

希望简化为

```c++
std::visit(make_visitor(
    [](int) { /* ... */},
    [](std::string) { /* ... */},
), v);
```

```c++
template<typename... Ts>
class Visitor : Ts...
{
public:
    Visitor(Ts&&... args)
        : Ts(args)... { }
};

template<typename... Ts>
decltype(auto) make_visitor(Ts&&... args)
{
    return Visitor<Ts...>(std::forward<Ts>(args)...);
}
```

原理，直接继承所有传入的`lambda`，获得重载的`operator()`版本。

这个版本事实上跑不了，因为函数重载要求函数都在同一个scope内，即使是在基类也不可以，补上`using`。

```c++
template<typename... Ts>
class Visitor : Ts...
{
public:
    Visitor(Ts&&... args)
        : Ts(std::forward<Ts>(args))... { }
    
    using Ts::operator()...;
};
```

另外一种传统实现是递归。

```c++
template<typename... Ts>
class Visitor;

template<typename T, typename... Ts>
class Visitor<T, Ts...> : public T, public Visitor<Ts...>
{
public:
    Visitor(T t, Ts&&... ts)
        : T(t), Visitor<Ts...>(std::forward<Ts>(ts)...) {}

    using T::operator();
    using Visitor<Ts...>::operator();
};

template<typename T>
class Visitor<T> : public T
{
public:
    Visitor(T&& t)
        : T(std::forward<T>(t)) {}

    using T::operator();
};
```
