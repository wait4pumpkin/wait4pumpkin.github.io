## 取第N个可变参数

```c++
// 取值，转tuple再get
template<std::size_t N, typename... Ts>
decltype(auto) get(Ts&&... ts) {
    return std::get<N>(std::forward_as_tuple(ts...));
}

// 取类型，转tuple再tuple_element
template<std::size_t N, typename... Ts> using nth_type =
    typename std::tuple_element<N, std::tuple<Ts...>>::type;

// tuple_element实现
template<std::size_t N, typename T>
struct tuple_element;
 
// recursive case
template<std::size_t N, typename Head, typename... Tail>
struct tuple_element<N, std::tuple<Head, Tail...>>
    : std::tuple_element<N - 1, std::tuple<Tail...>> { };
 
// base case
template<typename Head, typename... Tail>
struct tuple_element<0, std::tuple<Head, Tail...>> {
    typedef Head type;
};
```

## Reference

+ [template parameter packs access Nth type and Nth element](https://stackoverflow.com/questions/20162903/template-parameter-packs-access-nth-type-and-nth-element/29753388)
+ [std::tuple_element<std::tuple>](https://en.cppreference.com/w/cpp/utility/tuple/tuple_element)
