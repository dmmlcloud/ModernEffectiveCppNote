# Moving to Modern C++

## 条款七:区别使用()和{}创建对象

```cpp
Widget w1; //调用默认构造函数
Widget w2 = w1; //不是赋值运算符，调用拷贝构造函数
w1 = w2; //是一个赋值运算符，调用operator=函数
```

C++11 使用统一初始化,统一初始化是一个概念上的东西，而括号初始化是一个具体语法构型。
使用花括号，指定一个容器的元素变得很容易：

```cpp
std::vector<int> v{1,3,5}; //v包含1,3,5
```

括号初始化也能被用于为⾮静态数据成员指定默认初始值。C++11 允许"="初始化也拥有这种能⼒：

```cpp
class Widget{
...
private:
    int x{0}; //没问题，x初始值为0
    int y = 0; //同上
    int z(0); //错误！
}
```

不可拷贝的对象可以使用花括号初始化或者小括号初始化，但是不能使用"="初始化

```cpp
std::vector<int> ai1{0}; //没问题，x初始值为0
std::atomic<int> ai2(0); //没问题
std::atomic<int> ai3 = 0; //错误！
```

在 C++中这三种⽅式都被指派为初始化表达式，但是只有括号任何地⽅都能被使用，所以叫统一初始化
但是括号初始化不允许内置隐式的变窄转换：

```cpp
double x,y,z;
int sum1{x+y+z}; //错误！三个double的和不能用来初始化int类型的变量
int sum2(x + y +z); //可以（表达式的值被截为int）
int sum3 = x + y + z; //同上
```

C++规定任何能被决议为一个声明的东西必须被决议为声明。这个规则的副作用是让很多程序员备受折磨：当他们想创建一个使用默认构造函数构造的对象，却不小⼼变成了函数声明。

```cpp
Widget w1(10); //使用实参10调用Widget的一个构造函数
Widget w2(); //最令⼈头疼的解析！声明一个函数w2，返回Widget
// 使用花括号不会
Widget w3{}; //调用没有参数的构造函数构造对象
```

括号初始化的缺点是有时它有一些令⼈惊讶的⾏为。Item2 解释了当 auto 声明的变量使用花括号初始化，变量就会被推导为 std::initializer_list，尽管使用相同内容的其他初始化⽅式会产⽣正常的结果。所以，你越喜欢用 atuo，你就越不能用括号初始化。
在构造函数调用中，只要不包含 std::initializer_list 参数，那么花括号初始化和小括号初始化都会产⽣一
样的结果：

```cpp
class Widget {
public:
    Widget(int i, bool b); //未声明默认构造函数
    Widget(int i, double d); // std::initializer_list参数
…
};
Widget w1(10, true); // 调用构造函数
Widget w2{10, true}; // 同上
Widget w3(10, 5.0); // 调用第二个构造函数
Widget w4{10, 5.0}; // 同上

// 添加一个以std::initializer_list为参数的函数
// w2和w4将会使用新添加的构造函数构造，即使另一个⾮std::initializer_list构造函数对于实参是更好的选择
class Widget {
public:
    Widget(int i, bool b); // 同上
    Widget(int i, double d); // 同上
    Widget(std::initializer_list<long double> il); //新添加的
…
};
Widget w1(10, true); // 使用小括号初始化
//调用第一个构造函数
Widget w2{10, true}; // 使用花括号初始化
// 调用第三个构造函数
// (10 和 true 转化为long double)
Widget w3(10, 5.0); // 使用小括号初始化
// 调用第二个构造函数
Widget w4{10, 5.0}; // 使用花括号初始化
// 调用第三个构造函数
// (10 和 5.0 转化为long double)


// 甚⾄普通的构造函数和移动构造函数都会被std::initializer_list构造函数劫持：
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<long double> il);
    operator float() const; // convert to float
};
Widget w5(w4); // 使用小括号，调用拷贝构造函数
Widget w6{w4}; // 使用花括号，调用std::initializer_list构造函数
Widget w7(std::move(w4)); // 使用小括号，调用移动构造函数
Widget w8{std::move(w4)}; // 使用花括号，调用std::initializer_list构造函数

Widget w{10, 5.0}; //错误！要求变窄转换

// 只有当没办法把括号初始化中实参的类型转化为std::initializer_list时，编译器才会回到正常的函数决议流程中。
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<std::string> il);
…
};
// int, bool, double无法转化为string
Widget w1(10, true); // 使用小括号初始化，调用第一个构造函数
Widget w2{10, true}; // 使用花括号初始化，调用第一个构造函数
Widget w3(10, 5.0); // 使用小括号初始化，调用第二个构造函数
Widget w4{10, 5.0}; // 使用花括号初始化，调用第二个构造函数

class Widget {
public:
    Widget();
    Widget(std::initializer_list<int> il);
...
};
Widget w1; // 调用默认构造函数
Widget w2{}; // 同上
Widget w3(); // 最令⼈头疼的解析！声明一个函数
Widget w4({}); // 调用std::initializer_list
Widget w5{{}}; // 同上
```

std::vector 有一个⾮ std::initializer_list 构造函数允许你去指定容器的初始⼤小，以及使用一个值填满你的容器。
但它也有一个 std::initializer_list 构造函数允许你使用花括号⾥⾯的值初始化容器。如果你创建一个数值类型的 vector，然后你传递两个实参。把这两个实参放到小括号和放到花括号中是不同：

```cpp
std::vector<int> v1(10, 20); //使用⾮std::initializer_list
//构造函数创建一个包含10个元素的std::vector
//所有的元素的值都是20
std::vector<int> v2{10, 20}; //使用std::initializer_list
//构造函数创建包含两个元素的std::vector
//元素的值为10和20
```

- 第一，作为一个类库作者，你需要意识到如果你的一堆构造函数中重载过一个或者多个
  std::initializer_list，用⼾代码如果使用了括号初始化，可能只会看到你重载的 std::initializer_list 这一个版本的构造函数。
- 第二，作为一个类库使用者，你必须认真的在花括号和小括号之间选择一个来创建对象。

如果你是一个模板的作者，花括号和小括号创建对象就更麻烦了。通常不能知晓哪个会被使用。
举个例⼦，假如你想创建一个接受任意数量的参数，然后用它们创建一个对象。使用可变参数模板(variadic template )可以⾮常简单的解决：

```cpp
template<typename T, typename... Ts>
void doSomeWork(Ts&&... params) {
create local T object from params... …
}

T localObject(std::forward<Ts>(params)...); // 使用小括号
T localObject{std::forward<Ts>(params)...}; // 使用花括号

std::vector<int> v;
…
doSomeWork<std::vector<int>>(10, 20);
```

如果 doSomeWork 创建 localObject 时使用的是小括号，std::vector 就会包含 10 个元素。
如果 doSomeWork 创建 localObject 时使用的是花括号，std::vector 就会包含 2 个元素。
哪个是正确的？doSomeWork 的作者不知道，只有调用者知道。
这正是标准库函数 std::make_unique 和 std::make_shared（参⻅ Item21）⾯对的问题。

## Item7-remember

- 括号初始化是最⼴泛使用的初始化语法，它防⽌变窄转换，并且对于 C++最令⼈头疼的解析有天⽣的免疫性
- 在构造函数重载决议中，括号初始化尽最大可能与 std::initializer_list 参数匹配，即便其他构造函数看起来是更好的选择
- 对于数值类型的 std::vector 来说使用花括号初始化和小括号初始化会造成巨⼤的不同
- 在模板类选择使用小括号初始化或使用花括号初始化创建对象是一个挑战。

## 条款八:优先考虑 nullptr 而非 0 和 NULL

如果 C++发现在当前上下⽂只能使用指针，它会很不情愿的把 0 解释为指针，NULL 也是如此，但是 0 和 NULL 都不是指针类型，在作为重载函数的参数的时候，他们不会调用指针对应的函数

```cpp
void f(int); //三个f的重载函数
void f(bool);
void f(void*);
f(0); //调⽤f(int)而不是f(void*)
f(NULL); //可能不会被编译，⼀般来说调⽤f(int),绝对不会调⽤f(void*)
```

f(NULL)的不确定行为是由 NULL 的实现不同造成的
nullptr 的优点是它不是整型，它也不是⼀个指针类型，但是可以把它认为是通用类型的指针。nullptr 的真正类型是 std::nullptr_t，在⼀个完美的循环定义以后，std::nullptr_t 又被定义为 nullptr。
std::nullptr_t 可以转换为指向任何内置类型的指针。

```cpp
f(nullptr); //调⽤重载函数f的f(void*)版本
```

在使用 auto 时，nullptr 能够帮助我们确定类型

```cpp
auto result = findRecord( /* arguments */ );
if (result == 0) { // 不知道result是个整型还是指针类型
    …
}
auto result = findRecord( /* arguments */ );
if (result == nullptr) { // result一定是指针类型
    …
}
```

nullptr 在模板中更加有用

```cpp
int f1(std::shared_ptr<Widget> spw);    // 只能被合适的
double f2(std::unique_ptr<Widget> upw); // 已锁互斥量调
bool f3(Widget* pw);                    // ⽤
std::mutex f1m, f2m, f3m; // 互斥量f1m，f2m，f3m，各种⽤于f1，f2，f3函数
using MuxGuard = // C++11的typedef，参⻅Item9
std::lock_guard<std::mutex>;
…
// 非模板版本
{
    MuxGuard g(f1m);     // 为f1m上锁
    auto result = f1(0); // 向f1传递控制空指针
} // 解锁
…
{
    MuxGuard g(f2m);        // 为f2m上锁
    auto result = f2(NULL); // 向f2传递控制空指针
} // 解锁
…
{
    MuxGuard g(f3m);           // 为f3m上锁
    auto result = f3(nullptr); // 向f3传递控制空指针
} // 解锁

// 为减少代码重复性，使用模板
// c++14
template<typename FuncType, typename MuxType, typename PtrType>
decltype(auto) lockAndCall(FuncType func, MuxType& mutex, PtrType ptr) {
    MuxGuard g(mutex);
    return func(ptr);
}

auto result1 = lockAndCall(f1, f1m, 0); // 错误！ 0的类型被推断为int
…
auto result2 = lockAndCall(f2, f2m, NULL); // 错误！NULL会被推断为⼀个int或者类似int的类型
…
auto result3 = lockAndCall(f3, f3m, nullptr); // 没问题
```

## Item8-remember

- 优先考虑 nullptr 而⾮ 0 和 NULL
- 避免重载指针和整型

## 条款九:优先考虑别名声明而⾮ typedefs

对于复杂的类型可以使用 typedef 或是 using

```cpp
// 效果完全一样
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS; // c++98
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>; // c++11
```

当声明⼀个函数指针时别名声明更容易理解：

```cpp
// FP是⼀个指向函数的指针的同义词，它指向的函数带有int和const std::string&形参，不返回任何东
西
typedef void (*FP)(int, const std::string&); // typedef
//同上
using FP = void (*)(int, const std::string&); // 别名声明
```

特别的，别名声明可以被模板化但是 typedef 不能。

```cpp
// MyAlloc为自定义内存分配器
template<typename T>
using MyAllocList = std::list<T,MyAlloc<T>>;
MyAllocList<Widget> lw;

//使用typedef
template<typename T>
struct MyAllocList {
typedef std::list<T, MyAlloc<T>> type;
};
MyAllocList<Widget>::type lw;

// 如果需要在模板内使用
// 需要在typedef前面加上typename
// 这⾥MyAllocList::type使⽤了⼀个类型，这个类型依赖于模板参数T。因此MyAllocList::type是⼀个依赖类型，在C++很多讨⼈喜欢的规则中的⼀个提到必须要在依赖类型名前加上typename。

template<typename T>
class Widget {
private:
    typename MyAllocList<T>::type list;
    …
};

// 如果使用别名声明则不需要使用typename
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>; // as before
template<typename T>
class Widget {
private:
    MyAllocList<T> list;
…
};

```

当编译器在 Widget 的模板中看到 MyAllocList::type（使⽤ typedef 的版本），它不能确定那是⼀个类型的名称。因为可能存在 MyAllocList 的⼀个特化版本没有 MyAllocList::type。

C++11 在 type traits 中给了你⼀系列⼯具去实现类型转换，如果要使⽤这些模板请包含头⽂件<type_traits>

```cpp
std::remove_const<T>::type // 从const T中产出T
std::remove_reference<T>::type // 从T&和T&&中产出T
std::add_lvalue_reference<T>::type // 从T中产出T&
```

如果你在⼀个模板内部使⽤类型参数，你也需要在它们前⾯加上 typename。因为这些 type traits 是通过在 struct 内嵌套 typedef 来实现的。

```cpp
std::remove_const<T>::type // C++11: const T → T
std::remove_const_t<T> // C++14 等价形式
std::remove_reference<T>::type // C++11: T&/T&& → T
std::remove_reference_t<T> // C++14 等价形式
std::add_lvalue_reference<T>::type // C++11: T → T&
std::add_lvalue_reference_t<T> // C++14 等价形式
// 可以靠别名声明实现
template <class T>
using remove_const_t = typename remove_const<T>::type;
template <class T>
using remove_reference_t = typename remove_reference<T>::type;
template <class T>
using add_lvalue_reference_t = typename add_lvalue_reference<T>::type;
```

## Item9-remember

- typedef 不支持模板化，但是别名声明支持。
- 别名模板避免了使用"::type"后缀，而且在模板中使用 typedef 还需要在前面加上 typename
- C++14 提供了 C++11 所有类型转换的别名声明版本
