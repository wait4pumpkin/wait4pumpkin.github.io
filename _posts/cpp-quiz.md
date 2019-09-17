

## Bool Conversion

```c++
void foo(const std::string& in) { std::cout << in << '\n'; }
void foo(bool in) { std::cout << "bool: " << in << '\n';}

foo("Hello World");
```

这两个重载版本，最终会选择`bool`版本，因为`bool`转换是标准转换，优于自定义转换

> A prvalue of arithmetic, unscoped enumeration, pointer, or pointer to member type can be converted to a prvalue of type bool. A zero value, null pointer value, or null member pointer value is converted to false; any other value is converted to true….

> A standard conversion sequence is always better than a user-defined conversion sequence

对于`variant`来说，这个问题依然存在（P0608修正）

```c++
// 类型是bool
std::variant<std::string, bool, int> var { "Hello World" };

// 编译错误，多个narrowing conversions
std::variant<float, long, double> v = 0;
```

## Reference

+ [[Quick Case] Surprising Conversions of const char* to bool](https://www.bfilipek.com/2019/07/surprising-conversions-char-bool.html)
