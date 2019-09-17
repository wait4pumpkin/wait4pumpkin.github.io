# X Macro

基本形式，定义数据列表

```c++
#define LIST_OF_VARIABLES \
    X(value1) \
    X(value2) \
    X(value3)
```

数据列表只在一个地方定义，在多个地方通过定义`X`进行展开

```c++
#define X(name) int name;
    LIST_OF_VARIABLES
#undef X
```

```c++
void print_variables()
{
#define X(name) printf("%s = %d\n", #name, name);
    LIST_OF_VARIABLES
#undef X
}
```

相当于

```c++
int value1;
int value2;
int value3;

void print_variables()
{
    printf("%s = %d\n", "value1", value1);
    printf("%s = %d\n", "value2", value2);
    printf("%s = %d\n", "value3", value3);
}
```

## 参考

[X Macro](https://en.wikipedia.org/wiki/X_Macro)
