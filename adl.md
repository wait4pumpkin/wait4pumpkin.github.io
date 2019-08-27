# Argument-dependent lookup

## ADL触发条件

两个条件
+ function call（包含operator）（隐含要求，unqualified lookup找到的必须是function）
+ unqualified-id

如果使用了任何::-qualification，不会触发ADL

```c++
namespace A {
    struct A { operator int(); };
    void f(A);
}
namespace B {
    void f(int);
    void test() {
        A::A a;
        f(a);     // ADL, calls A::f(A)
        B::f(a);  // no ADL, has qualification, calls B::f(int)
        (f)(a);   // no ADL, not unqualified-id, calls B::f(int)
    }
}
```

如果不是unqualified-id，例如primary-expression，如`(f)()`，也不会触发ADL

```c++
namespace A {
    struct A { operator int(); };
    void f(A);
}
namespace B {
    void f(int);
    void f(double);
    void test() {
        A::A a;
        void (*fp)(int) = f;  // OK, no ADL
        void (*gp)(A::A) = f; // ERROR, not function call, no ADL, fails to find A::f
        f(a);     // ADL, calls A::f(A)
        (&f)(a);  // no ADL, not unqualified-id, calls B::f(int)
        (f)(a);   // no ADL, not unqualified-id, calls B::f(int)
    }
}
```

如果通过普通的unqualified lookup找到的符号并不是一个function（lambda也不属于function），则不会触发ADL

```c++
namespace A {
    struct A { operator int(); };
    void f(A);
    void g(A);
    void h(A);
    int i(A);
    int j(A);
}
namespace B {
    void f(int);
    auto h = [](int) {};
    using i = int;
    void test() {
        A::A a;
        f(a);           // ADL, calls A::f(A)
        g(a);           // ADL, calls A::g(A)
        h(a);           // no ADL: lookup found B::h which is not a function
        int ia = i(a);  // no ADL: lookup found B::i which is not a function
        int j = j(a);   // no ADL, and ERROR! lookup found local variable j
    }
}
```

## ADL搜索范围

ADL只考虑参数的最终类型信息，中间推导过程的namespace全部忽略

```c++
namespace A {
    struct A {};
}
namespace B {
    using T = A::A;
}
namespace C {
    B::T c;
}
namespace C {
    void test() {
        // 最终类型是A::A，ADL不会搜索和namespace B，也不会搜索namespace C
        f(C::c);  // HERE
    }
}
```

ADL不考虑函数模板参数

```c++
namespace A {
    struct A { operator int(); };
    struct X {};

    template<class T>
    void f(int);
}
namespace B {
    template<class T>
    void f();

    void test() {
        A::A a;
        f<A::X>();   // OK, ADL doesn't consider A::f, calls B::f
        f<A::X>(a);  // OK, ADL considers A::f because of A::A, calls A::f
        f<A::X>(42); // ERROR: ADL doesn't consider A::f
    }
}
```

具体搜索过程是，从每个参数（没有固定顺序），找出相关的类型和namespace，再从这些空间选择最佳匹配函数

+ Primitive类型（如`int`），没有关联类型
+ `NS::SomeClass`（或者`NS::SomeClass *`，`NS::SomeClass &`），关联类型`NS::SomeClass`，关联namespace `NS`
+ `NN::NS::SomeClass`，关联类型自身`NN::NS::SomeClass`，关联namespace `NN::NS`（`NN`不是关联namespace）
+ `SomeClass::NestedClassOrEnum`，关联类型自身`SomeClass::NestedClassOrEnum`和`SomeClass`
+ `NA::A (*)(NB::B, NC::C)`，关联类型`NA::A`、`NB::B`和`NC::C`，关联namespace `NA`、`NB`和` NC`
+ `NS::SomeTemplate<NA::A, NB::B>`，关联类型自身、`NA::A`和`NB::B`，关联namespace `NS`、`NA`和`NB`
+ `NS::SomeClass::SomeNestedTemplate<NA::A>`，关联类型自身、`NA::A`和`NS::SomeClass`，关联 namespaces `NS`和`NA`
+ `NA::A`（继承自NB::B，即使是private）, 关联类型`NA::A`和`NB::B`，关联namespace `NA`和`NB`

搜索过程是非贪婪的，就是说`class A::B::C`，只与`A::B`相关联，而与`A`无关

对于相关类型的搜索，只搜索`friend`，不搜索成员函数（包括`static`）

```c++
namespace N {
    struct A {
        enum E { E0 };

        friend void f(E);
        static void g(E);
    };
}

namespace My {
    void f(int);
    void g(int);
    void test() {
        N::A::E e;
        f(e);  // ADL considers N::f (friend of N::A)
        g(e);  // ADL does not consider N::A::g
    }
}
```

ADL会忽略带有explicit namespace-qualification的friend function

```c++
namespace Unrelated {
    void f(int);
}

namespace NN {
    void f(int);
    namespace NA {
        struct A {
            enum E : int { E0 };

            friend void f(int);
            friend void NN::f(int);
            friend void Unrelated::f(int);
        };
    }
}

namespace B {
    void test() {
        NN::NA::A::E e;
        f(e);  // OK: ADL considers NA::f which is an unqualified
               // ("namespace-scope") friend of NA::A, but not
               // the other two friends
    }
}
```

ADL只会考虑函数声明，非函数会忽略（如果unqualified lookup找到非函数则根本不用使用ADL）

```c++
namespace A {
    struct A { operator int() const; };
    auto f = [](A, int) {};
    void g(A, int);
    void h(A, int);
}
namespace B {
    struct B { operator int() const; };
    void f(int, B);
    using g = int;
    void h(int, B);
}

namespace C {
    void f(int, int);
    void g(int, int);
    auto h = [](int, int) {};
    void test() {
        A::A a;
        B::B b;
        f(a, b);  // OK: ADL ignores the non-function A::f
        g(a, b);  // OK: ADL ignores the non-function B::g
        h(a, b);  // OK: no ADL
    }
}
```

## 应用

```c++
#include <stdio.h>

namespace A {
    struct A {};
    void call(void (*f)()) {
        f();
    }
}

void f() {
    puts("Hello world");
}

void f(A::A);  // UNUSED

int main() {
    call(f);
}
```

如果没有添加UNUSED的重载版本，会编译出错，无法找到`call`。加了这个版本，事实上是触发了ADL，增加了关联的namespace A。

```c++
namespace unexpected {
    struct SomeType;
    template<class T> void helper(T&&);
}
namespace A = unexpected;

namespace expected {
    void helper(const basic_type<A::SomeType>&);

    void test() {
        helper( basic_type<A::SomeType>{} );
    }
}
```

`basic_type<A::SomeType>`触发了ADL，添加了关联namespace unexpected，最终调用了`unexpected::helper`。

```c++
template<class T>
struct Enhanced {
    struct type {};
};

template<class T>
using enhanced_type = typename Enhanced<T>::type;

namespace unexpected {
    struct SomeType;
    template<class T> void helper(T&&);
}
namespace A = unexpected;

namespace expected {
    void helper(const enhanced_type<A::SomeType>&);

    void test() {
        helper( enhanced_type<A::SomeType>{} );
    }
}
```

此处`enhanced_type<A::SomeType>`同样会触发ADL，但是因为ADL的规则是非贪婪的，namespace unexpected并不关联，相当于抑制了ADL。


## Reference

+ [What is ADL?](https://quuxplusone.github.io/blog/2019/04/26/what-is-adl/)
+ [ADL insanity](https://quuxplusone.github.io/blog/2019/04/08/adl-insanity/)
+ [How hana::type<T> “disables ADL”](https://quuxplusone.github.io/blog/2019/04/09/adl-insanity-round-2/)
