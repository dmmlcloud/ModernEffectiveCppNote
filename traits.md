# Traits
在编译期获得类型信息，Traits时一种共同遵守的协议，对内置类型和自定义类型必须有一样好的表现。
traits信息必须位于类型自身之外，标准技术时将它放进一个template以及多个特化的版本中
## Type Traits
- 用于内置类型判断
```cpp
struct true_type {
    static constexpr bool value = true;
};

struct false_type {
    static constexpr bool value = false;
};

template <typename>
struct is_void : false_type {};

template <>
struct is_void<void> : true_type {};

template <typename>
struct is_floating_point : false_type {};

template <>
struct is_floating_point<float> : true_type {};

template <>
struct is_floating_point<double> : true_type {};

template <>
struct is_floating_point<long double> : true_type {};
```
- 可以处理类型，去除const或引用等
```cpp
template <typename T>
struct remove_const {
    using type = T;
};

template <typename T>
struct remove_const<const T> {
    using type = T;
};

template <typename T>
struct remove_reference {
    using type = T;
};

template <typename T>
struct remove_reference<T &> {
    using type = T;
};

template <typename T>
struct remove_reference<T &&> {
    using type = T;
};
```
- 可以进行类型选择，当true时为类型T1，否则为类型T2
```cpp
template <bool judge, typename T1, typename T2>
struct conditional {
    using type = T1;
};

template <typename T1, typename T2>
struct conditional<false, T1, T2> {
    using type = T2;
};
```
## stl_iterator
### advance
- 可以用于可以随机访问的容器，使迭代器向前移动n个
```cpp
template<typename IterT, typename DistT>
void advance (IterT& iter, Dist d);
```
### 5种迭代器分类
- input: 只能向前移动，一次一步，只能读取一次，不能修改
- ouput: 只能向前移动，一次一步，只能修改一次，不能读取
- forward: 只能向前移动，能读写多次（一些库中的linked list(单向链表)）
- bidirectional: 可以前后移动，读写多次（set, multiset, map, multimap）
- random access: 常量时间内移动任意距离，读写多次（vector, deque, string）

```cpp
struct input_iterator_tag{};
struct output_iterator_tag{};
struct forward_iterator_tag: public input_iterator_tag{};
struct bidirectional_iterator_tag: public forward_iterator_tag{};
struct random_accesst_iterator_tag: public bidirectional_iterator_tag{};
```
### advance实现
```cpp
// only random access iterator can use
template <typename IterT, typename DistT>
void advance(IterT& iter, Dist d)
{
    if(iter is a random access iterator) {
        iter += d;
    } else {
        if(d >= 0) { while(d--) ++iter; }
        else { while(d++) --iter; }
    }
}
```

## Iterator Traits
针对于iterator的被命名为iterator_traits:
- 对于每种类型的iterT，iterator_traits<iterT>在内部声明某个typedef名为iterator_category，用于确认iterT的分类
```cpp
template <typename IterT>
struct iterator_traits {
    typedef typename IterT::iterator_category iterator_category; // IterT::iterator_category iterator表现自己的类型
    typedef typename IterT::value_type value_type; // IterT中值的类型
}
```
同时需要对指针进行特化，因为指针内不能嵌套typedef
```cpp
template <typename IterT>
struct iterator_traits<IterT*> {
    typedef random_access_iterator_tag iterator_category; // 指针类型的迭代器和random_access类似
```
- 用户自定义迭代器类型，必须嵌套一个typedef，名为iterator_category，用来确认适当的卷标结构
```cpp
template<type name T>
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
        typedef T value_type;
    }
}
```
- 对需要对不同迭代器行为进行不同操作的函数进行重载
```cpp
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, Dist d, std::random_access_iterator_tag){
    iter += d;
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, Dist d, std::bidirectional_iterator_tag){
    if(d >= 0) { while (d--) ++iter; }
    else { while (d++) --iter ;}
}

template<typename IterT, typename DistT>
void doAdvance(IterT& iter, Dist d, std::input_iterator_tag){
    if(d < 0) {
        throw std::out_of_range("Negative distance");
    }
    while(d--) ++iter;
}

// 调用
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d)
{
    doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category() );
}
```