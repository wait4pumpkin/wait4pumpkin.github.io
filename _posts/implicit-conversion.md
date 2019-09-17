# 隐式转换

## 浮点整型转换

+ 浮点类型的纯右值可隐式转换成任何整数类型的纯右值。截断小数部分，若结果不能适应到目标类型中。
+ 整数或无作用域枚举类型的纯右值可转换成任何浮点类型的纯右值。若不能正确表示该值，则选择与之最接近的较高值还是最接近的较低值是实现定义的，不过若支持 IEEE，则舍入默认为到最接近。若其值不能适应到目标类型中，则行为未定义。

## lambda转换

captureless的lambda可以隐式转换为函数指针。如果禁止隐式转换的环境下（如模板），可以通过unvary +转换（因为lambda没定义+，而函数指针有）。

```c++
template<typename R, typename A>
void foo(R (*fptr)(A))
{
    std::puts(__PRETTY_FUNCTION__);
}
foo( [](double x) { return int(x); } );  // error
foo(+[](double x) { return int(x); } );  // compiles
```
