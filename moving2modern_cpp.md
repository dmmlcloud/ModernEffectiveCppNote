# Moving to Modern C++

## Item7:区别使用()和{}创建对象

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

括号初始化也能被用于为非静态数据成员指定默认初始值。C++11 允许"="初始化也拥有这种能力：

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

在 C++中这三种方式都被指派为初始化表达式，但是只有括号任何地方都能被使用，所以叫统一初始化
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
Widget w2(); //最令人头疼的解析！声明一个函数w2，返回Widget
// 使用花括号不会
Widget w3{}; //调用没有参数的构造函数构造对象
```

括号初始化的缺点是有时它有一些令人惊讶的行为。Item2 解释了当 auto 声明的变量使用花括号初始化，变量就会被推导为 std::initializer_list，尽管使用相同内容的其他初始化方式会产生正常的结果。所以，你越喜欢用 atuo，你就越不能用括号初始化。
在构造函数调用中，只要不包含 std::initializer_list 参数，那么花括号初始化和小括号初始化都会产生一
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
// w2和w4将会使用新添加的构造函数构造，即使另一个非std::initializer_list构造函数对于实参是更好的选择
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
Widget w3(); // 最令人头疼的解析！声明一个函数
Widget w4({}); // 调用std::initializer_list
Widget w5{{}}; // 同上
```

std::vector 有一个非 std::initializer_list 构造函数允许你去指定容器的初始大小，以及使用一个值填满你的容器。
但它也有一个 std::initializer_list 构造函数允许你使用花括号里面的值初始化容器。如果你创建一个数值类型的 vector，然后你传递两个实参。把这两个实参放到小括号和放到花括号中是不同：

```cpp
std::vector<int> v1(10, 20); //使用非std::initializer_list
//构造函数创建一个包含10个元素的std::vector
//所有的元素的值都是20
std::vector<int> v2{10, 20}; //使用std::initializer_list
//构造函数创建包含两个元素的std::vector
//元素的值为10和20
```

- 第一，作为一个类库作者，你需要意识到如果你的一堆构造函数中重载过一个或者多个
  std::initializer_list，用户代码如果使用了括号初始化，可能只会看到你重载的 std::initializer_list 这一个版本的构造函数。
- 第二，作为一个类库使用者，你必须认真的在花括号和小括号之间选择一个来创建对象。

如果你是一个模板的作者，花括号和小括号创建对象就更麻烦了。通常不能知晓哪个会被使用。
举个例子，假如你想创建一个接受任意数量的参数，然后用它们创建一个对象。使用可变参数模板(variadic template )可以非常简单的解决：

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
这正是标准库函数 std::make_unique 和 std::make_shared（参见 Item21）面对的问题。

## Item7-remember

- 括号初始化是最⼴泛使用的初始化语法，它防止变窄转换，并且对于 C++最令人头疼的解析有天生的免疫性
- 在构造函数重载决议中，括号初始化尽最大可能与 std::initializer_list 参数匹配，即便其他构造函数看起来是更好的选择
- 对于数值类型的 std::vector 来说使用花括号初始化和小括号初始化会造成巨大的不同
- 在模板类选择使用小括号初始化或使用花括号初始化创建对象是一个挑战。

## Item8:优先考虑 nullptr 而非 0 和 NULL

如果 C++发现在当前上下⽂只能使用指针，它会很不情愿的把 0 解释为指针，NULL 也是如此，但是 0 和 NULL 都不是指针类型，在作为重载函数的参数的时候，他们不会调用指针对应的函数

```cpp
void f(int); //三个f的重载函数
void f(bool);
void f(void*);
f(0); //调用f(int)而不是f(void*)
f(NULL); //可能不会被编译，一般来说调用f(int),绝对不会调用f(void*)
```

f(NULL)的不确定行为是由 NULL 的实现不同造成的
nullptr 的优点是它不是整型，它也不是一个指针类型，但是可以把它认为是通用类型的指针。nullptr 的真正类型是 std::nullptr_t，在一个完美的循环定义以后，std::nullptr_t 又被定义为 nullptr。
std::nullptr_t 可以转换为指向任何内置类型的指针。

```cpp
f(nullptr); //调用重载函数f的f(void*)版本
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
bool f3(Widget* pw);                    // 用
std::mutex f1m, f2m, f3m; // 互斥量f1m，f2m，f3m，各种用于f1，f2，f3函数
using MuxGuard = // C++11的typedef，参见Item9
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
auto result2 = lockAndCall(f2, f2m, NULL); // 错误！NULL会被推断为一个int或者类似int的类型
…
auto result3 = lockAndCall(f3, f3m, nullptr); // 没问题
```

## Item8-remember

- 优先考虑 nullptr 而非 0 和 NULL
- 避免重载指针和整型

## Item9:优先考虑别名声明而非 typedefs

对于复杂的类型可以使用 typedef 或是 using

```cpp
// 效果完全一样
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS; // c++98
using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>; // c++11
```

当声明一个函数指针时别名声明更容易理解：

```cpp
// FP是一个指向函数的指针的同义词，它指向的函数带有int和const std::string&形参，不返回任何东
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
// 这里MyAllocList::type使用了一个类型，这个类型依赖于模板参数T。因此MyAllocList::type是一个依赖类型，在C++很多讨人喜欢的规则中的一个提到必须要在依赖类型名前加上typename。

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

当编译器在 Widget 的模板中看到 MyAllocList::type（使用 typedef 的版本），它不能确定那是一个类型的名称。因为可能存在 MyAllocList 的一个特化版本没有 MyAllocList::type。

C++11 在 type traits 中给了你一系列⼯具去实现类型转换，如果要使用这些模板请包含头⽂件<type_traits>

```cpp
std::remove_const<T>::type // 从const T中产出T
std::remove_reference<T>::type // 从T&和T&&中产出T
std::add_lvalue_reference<T>::type // 从T中产出T&
```

如果你在一个模板内部使用类型参数，你也需要在它们前面加上 typename。因为这些 type traits 是通过在 struct 内嵌套 typedef 来实现的。

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

## Item10:优先考虑限域枚举而非未限域枚举

在 enum 作用域中声明的枚举名所在的作用域也包括 enum 本身(未限域枚举)

```cpp
enum Color { black, white, red }; // black, white, red 和
                                  // Color一样都在相同作用域
auto white = false; // 错误! white早已在这个作用
                    // 域中存在
```

在 C++11 中它们有一个相似物，限域枚举(scoped enum)，它不会导致枚举名泄漏:

```cpp
enum class Color { black, white, red }; // black, white, red
// 限制在Color域内
auto white = false; // 没问题，同样域内没有这个名字
Color c = white; //错误，这个域中没有white
Color c = Color::white; // 没问题
auto c = Color::white; // 也没问题（也符合条款5的建议）
```

通过 enum class 声明，也被称为枚举类
在限域枚举的作用域内，枚举名是强类型，而在非限域枚举中枚举名会隐式转化为整型

```cpp
enum Color { black, white, red }; // 未限域枚举
std::vector<std::size_t> // func返回x的质因子
primeFactors(std::size_t x);
Color c = red;
…
if (c < 14.5) { // Color与double比较 (!)
    auto factors = // 计算一个Color的质因子(!)
        primeFactors(c);
…
}

enum class Color { black, white, red }; // Color现在是限域枚举
Color c = Color::red; // 和之前一样，只是
… // 多了一个域修饰符
if (c < 14.5) { // 错误！不能比较
// Color和double
    auto factors = // 错误! 不能向参数为std::size_t的函数
    primeFactors(c); // 传递Color参数
    …
}

// 可以通过强制类型转换使用
if (static_cast<double>(c) < 14.5) { // 奇怪的代码，但是
// 有效
    auto factors = // suspect, but
    primeFactors(static_cast<std::size_t>(c)); // 能通过编译
…
}

```

限域枚举可以前置声明

```cpp
enum Color; // 错误！需要一些限制
enum class Color; // 没问题
```

因为在 C++中所有的枚举都有一个由编译器决定的整型的基础类型。为了高效使用内存，编译器通常在确保能包含所有枚举值的前提下为枚举选择一个最小的基础类型。在
一些情况下，编译器将会优化速度，舍弃大小，这种情况下它可能不会选择最小的基础类型，而是选择对优化大小有帮助的类型。为此，C++98 只支持枚举定义（所有枚举名全部列出来）；枚举声明是不被允许的。这使得编译器能为之前使用的每一个枚举选择一个基础类型

不能前置声明会增加编译依赖

```cpp
enum Status { good = 0,
failed = 1,
incomplete = 100,
corrupt = 200,
audited = 500, // 新增的枚举项
indeterminate = 0xFFFFFFFF
};
```

可能整个系统都得重新编译，即使只有一个子系统或者一个函数使用了新添加的枚举名，如果使用前置声明，即使 Status 的定义发生改变，包含这些声明的头⽂件也不会重新编译。而且如果 Status 添加一个枚举名（比如添加一个 audited），continueProcessing 的行为不受影响（因为 continueProcessing 没有使用这个新添加的 audited），continueProcessing 也不需要重新编译。

```cpp
enum class Status; // forward declaration
void continueProcessing(Status s); // use of fwd-declared enum
```

限域枚举的基础类型总是已知的（默认 int），枚举的基础类型都是可以指定的

```cpp
enum class Status; // 基础类型是int
enum class Status: std::uint32_t; // Status的基础类型
                                    // 是std::uint32_t
                                    // (需要包含 <cstdint>)
enum Color: std::uint8_t; // 为非限域枚举Color指定
                            // 基础为
                            // std::uint8_t
// 也可以放在定义处
enum class Status: std::uint32_t {
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    audited = 500,
    indeterminate = 0xFFFFFFFF
};
```

在获取 C++11 tuples 中的字段的时候，非限域枚举是有用的，因为需要他转化为整型

```cpp
using UserInfo = // 类型别名，参见Item 9
std::tuple<std::string, // 名字
            std::string, // email地址
            std::size_t> ; // 声望
UserInfo uInfo; // tuple对象
…
// 很难记住每一位对应的含义，所以使用enum
auto val = std::get<1>(uInfo); // 获取第一个字段
enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
…
auto val = std::get<uiEmail>(uInfo); // 获取用户email

// 而如果使用限域枚举
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo; // as before
…
auto val =
std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>
(uInfo);
```

对于限域枚举可以封装一个强制类型转换的函数，std::get 是⼀个模板（函数），需要你给出⼀个 std::size_t 值的模板实参（注意使用 <> 而不是 () ），因此将枚举名变换为 std::size_t 值会发⽣在编译期，所以必须是⼀个 constexpr 模板函数。

```cpp
template<typename E> // C++14
constexpr decltype(auto)
toUType(E enumerator) noexcept
{
    return static_cast<std::underlying_type_t<E>>(enumerator);
    // underlying_type_t 返回的是枚举类型的基础类型
}
// 所以可以写成
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

## Item10-remember

- C++98 的枚举即非限域枚举
- 限域枚举的枚举名仅在 enum 内可见。要转换为其它类型只能使用 cast。
- 非限域/限域枚举都支持基础类型说明语法，限域枚举基础类型默认是 int 。非限域枚举没有默认基础类型。
- 限域枚举总是可以前置声明。非限域枚举仅当指定它们的基础类型时才能前置。

## Item11:优先考虑使用deleted函数而⾮使用未定义的私有声明
有时C++会给你⾃动声明⼀些函数，如果你想防⽌客户调用这些函数，事情就不那么简单了。

**在C++98中防⽌调用这些函数的⽅法是将它们声明为私有成员函数。**

举个例⼦，在C++ 标准库iostream继承链的顶部是模板类 basic_ios 。所有 istream 和 ostream 类都继承此类(直接或者间接)。拷⻉istream 和 ostream 是不合适的。
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …
private:
    basic_ios(const basic_ios& ); // not defined
    basic_ios& operator=(const basic_ios&); // not defined
};
```
**在C++11中有⼀种更好的⽅式，只需要使用相同的结尾：用 = delete 将拷⻉构造函数和拷⻉赋值运算符标记为 deleted 函数：**
```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …
    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    …
};
```
deleted 函数不能以任何⽅式被调用，即使你在成员函数或者友元函数⾥⾯调用deleted 函数也不能通过编译。

通常， deleted 函数被声明为public而不是private.这也是有原因的：当客⼾端代码试图调用成员函数，C++会在检查 deleted 状态前检查它的访问性，所以当客⼾端代码调用⼀个私有的 deleted 函数，⼀些编译器只会给出该函数是private的错误，而没有诸如该函数被 deleted 修饰的错误。

deleted 函数还有⼀个重要的优势是任何函数都可以标记为 deleted。

C++有沉重的C包袱，使得含糊的、能被视作数值的任何类型都能隐式转换为 int ，但是有⼀些调用可能是没有意义的：
```cpp
bool isLucky(int number);
if (isLucky('a')) … // 字符'a'是幸运数？
if (isLucky(true)) … // "true"是?
if (isLucky(3.5)) … // 难道判断它的幸运之前还要先截尾成3？
bool isLucky(char) = delete; // 拒绝char
bool isLucky(bool) = delete; // 拒绝bool
bool isLucky(double) = delete; // 拒绝float和double，在传入float时会转为double（优先）或int
```
另⼀个 deleted 函数用武之地（private成员函数做不到的地⽅）是禁⽌⼀些模板的实例化。
假如你要求⼀个模板仅⽀持原⽣指针：
```cpp
template<typename T>
void processPointer(T* ptr);
```
在指针的世界⾥有两种特殊情况。⼀是 void* 指针，因为没办法对它们进⾏解引用，或者加加减减等。另⼀种指针是 char* ，因为它们通常代表C⻛格的字符串，而不是正常意义下指向单个字符的指针。这两种情况要特殊处理，在 processPointer 模板⾥⾯，我们假设正确的函数应该拒绝这些类型。也即是说， processPointer 不能被 void* 和 char* 调用。
```cpp
template<>
void processPointer<void>(void*) = delete;
template<>
void processPointer<char>(char*) = delete;
template<>
void processPointer<const void>(const void*) = delete;
template<>
void processPointer<const char>(const char*) = delete;
// 如果你想做得更彻底⼀些，你还要删除 const volatile void* 和 const volatile char* 重载版本，另外还需要⼀并删除其他标准字符类型的重载版本： std::wchar_t , std::char16_t 和 std::char32_t 。
```
如果的类⾥⾯有⼀个函数模板，你可能想用 private （经典的C++98惯例）来禁⽌这些函数模板实例化，但是不能这样做，因为不能给特化的模板函数指定⼀个不同（于函数模板）的访问级别。
```cpp
class Widget {
public:
…
    template<typename T>
    void processPointer(T* ptr)
    { … }
private:
    template<> // 错误！
    void processPointer<void>(void*);
};
// delete 不会出现这个问题，因为它不需要⼀个不同的访问级别，且他们可以在类外被删除
class Widget {
public:
    …
    template<typename T>
    void processPointer(T* ptr)
    { … }
…
};
template<>
void Widget::processPointer<void>(void*) = delete; // 还是public，但是已经被删除了
```
## Item11-remember
- ⽐起声明函数为private但不定义，使用delete函数更好
- 任何函数都能 delete ，包括⾮成员函数和模板实例
- 函数模板意指未特化前的源码，模板函数则倾向于模板实例化后的函数

## Item12:使用override声明重载函数
虚函数重写：
```cpp
class Base {
public:
    virtual void doWork(); // 基类虚函数
    …
};
class Derived: public Base {
public:
    virtual void doWork(); // 重写Base::doWork(这⾥"virtual"是可以省略的)
        …
};
std::unique_ptr<Base> upb = // 创建基类指针
std::make_unique<Derived>(); // 指向派⽣类对象

… 
upb->doWork(); // 通过基类指针调用doWork
                // 实际上是派⽣类的doWork
                // 函数被调用
```
要想重写⼀个函数，必须满⾜下列要求：
- 基类函数必须是 virtual
- 基类和派⽣类函数名必须完全⼀样（除⾮是析构函数
- 基类和派⽣类函数参数必须完全⼀样
- 基类和派⽣类函数常量性(constness)必须完全⼀样
- 基类和派⽣类函数的返回值和异常说明(exception specifications)必须兼容
- 函数的引用限定符（reference qualifiers）必须完全⼀样。它可以限定成员函数只能用于左值或者右值。成员函数不需要 virtual 也能使用它们：（C++11）
```cpp
class Widget {
public:
    …
    void doWork() &; //只有*this为左值的时候才能被调用
    void doWork() &&; //只有*this为右值的时候才能被调用
};
…
Widget makeWidget(); // ⼯⼚函数（返回右值）
Widget w; // 普通对象（左值）
…
w.doWork(); // 调用被左值引用限定修饰的Widget::doWork版本
            // (即Widget::doWork &)
makeWidget().doWork(); // 调用被右值引用限定修饰的Widget::doWork版本
                        // (即Widget::doWork &&)
```
这么多的重写需求意味着哪怕⼀个小小的错误也会造成巨⼤的不同:
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};
class Derived: public Base {
public:
    virtual void mf1(); // const限定不同
    virtual void mf2(unsigned int x); // 参数不同
    virtual void mf3() &&; // 引用限定不同
    void mf4() const; // 非virtual
};
```
上述代码在编译时可能没有warning

由于正确声明派⽣类的重写函数很重要，但很容易出错，C++11提供⼀个⽅法让你可以显式的将派⽣类函数指定为应该是基类重写版本：将它声明为 override：
```cpp
class Derived: public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    virtual void mf4() const override;
};
```
代码不能编译，当然了，因为这样写的时候，编译器会抱怨所有与重写有关的问题，这也是你想要的，以及为什么要在所有重写函数后⾯加上 override。

正确写法：
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
};
class Derived: public Base {
public:
    virtual void mf1() const override;
    virtual void mf2(int x) override;
    virtual void mf3() & override;
    void mf4() const override; // 可以添加virtual，但不是必要
};
```
如果你考虑修改修改基类虚函数的函数签名， override 还可以帮你评估后果。如果派⽣类全都用上 override ，你可以只改变基类函数签名，重编译系统，再看看你造成了多⼤的问题（即，多少派⽣类不能通过编译），然后决定是否值得如此⿇烦更改函数签名。没有重写，你只能寄希望于完善的单元测试，因为，正如我们所⻅，派⽣类虚函数本想重写基类，但是没有，编译器也没有探测并发出诊断信息。

C++11引⼊了两个上下⽂关键字, override 和 final （向虚函数添加 final 可以防⽌派⽣类重写。 final 也能用于类，这时这个类不能用作基类）。

这两个关键字的特点是它们是保留的，它们只是位于特定上下⽂才被视为关键字。对于 override ，它只在成员函数声明结尾处才被视为关键字。这意味着如果你以前写的代码⾥⾯已经用过override这个名字，那么换到C++11标准你也⽆需修改代码：
```cpp
class Warning { // potential legacy class from C++98
public:
    …
    void override(); // C++98和C++11都合法
};
```
下面介绍成员函数的引用限定的使用场景：
假设我们的 Widget 类有⼀个 std::vector 数据成员，我们提供⼀个函数让客⼾端可以直接访问它：

```cpp
class Widget {
public:
    using DataType = std::vector<double>; // 参⻅Item
    …
    DataType& data() { return values; }
…
private:
    DataType values;
};

Widget w;
…
auto vals1 = w.data(); // 拷⻉w.values到vals1
//Widget::data函数的返回值是⼀个左值引用（准确的说是 std::vector<double>& ）,因为左值引用是左值， vals1 从左值初始化，因此它由 w.values 拷⻉构造而得
Widget makeWidget();
// 使用makeWidget 返回的 std::vector 初始化⼀个变量：
auto vals2 = makeWidget().data(); // 拷⻉Widget⾥⾯的值到vals2
// Widgets::data 返回的是左值引用，还有，左值引用是左值。所以，我们的对象(vals2)⼜得从Widget⾥的values拷⻉构造，但Widget 是 makeWidget 返回的临时对象（即右值），所以将其中的 std::vector 进⾏拷⻉纯属浪费。最好是移动，但是因为 data 返回左值引用，C++的规则要求编译器不得不⽣成⼀个拷⻉。
// 现在就可以使用引用限定写⼀个重载函数来达成这⼀⽬的：
class Widget {
public:
    using DataType = std::vector<double>;
    …
    DataType& data() & // 对于左值Widgets,
    { return values; } // 返回左值
    DataType data() && // 对于右值Widgets,
    { return std::move(values); } // 返回右值
    …
private:
    DataType values;
};

auto vals1 = w.data(); //调用左值重载版本的Widget::data，拷⻉构造vals1
auto vals2 = makeWidget().data(); //调用右值重载版本的Widget::data, 移动构造vals2
```
## Item12-remember
- 为重载函数加上 override
- 成员函数限定让我们可以区别对待左值对象和右值对象（即 *this）

## 条款 13:优先考虑const_iterator而非iterator
STL const_iterator等价于指向常量的指针，它们都指向不能被修改的值。
但是在C++98中，标准库对const_iterator的⽀持不是很完整。⾸先不容易创建它们，其次就算你有了它，它的使用也是受限的。
```cpp
typedef std::vector<int>::iterator IterT; // typetypedef
std::vector<int>::const_iterator ConstIterT; // defs
std::vector<int> values;
…
ConstIterT ci =
std::find(static_cast<ConstIterT>(values.begin()), // cast
            static_cast<ConstIterT>(values.end()), // cast
            1983);
values.insert(static_cast<IterT>(ci), 1998); // 可能⽆法通过编译，原因⻅下
```
之所以 std::find 的调用会出现类型转换是因为在C++98中values是⾮常量容器，没办法简简单单的从⾮常量容器中获取const_iterator，严格来说类型转换不是必须的，因为用其他⽅法获取const_iterator也是可以的（⽐如你可以把values绑定到常量引用上，然后再用这个变量代替values），但不管怎么说，从⾮常量容器中获取const_iterator的做法都有点别扭。

在获得了const_iterator，因为C++98中，插⼊操作的位置只能由iterator指定，const_iterator是不被接受的，所以在上述代码中将const_iterator转换为iterat的，因为向insert传⼊const_iterator不能通过编译。

但上⾯的代码依然⽆法编译，因为没有⼀个可移植的从const_iterator到iterator的⽅法，即使使用 static_cast 也不⾏，甚⾄reinterpret_cast也不行。

**所以const_iterator在C++98中会有很多问题**

这些都在C++11中改变了，现在const_iterator即容易获取⼜容易使用。容器的成员函数cbegin 和 cend 产出const_iterator，甚⾄对于⾮常量容器，那些之前只使用iterator指⽰位置的STL成员函数也可以使用const_iterator了。使用C++11 const_iterator重写C++98使用iterator的代码也稀松平常：
```cpp
std::vector<int> values; // 和之前⼀样
…
auto it = // 使用cbegin
std::find(values.cbegin(),values.cend(), 1983); // 和cend
values.insert(it, 1998);
```
唯⼀⼀个C++11对于const_iterator⽀持不⾜（C++14⽀持但是C++11的时候还没）的情况是：当你想写最⼤程度通用的库，并且这些库代码为⼀些容器和类似容器的数据结构提供⾮成员函数begin、end（以及cbegin，cend，rbegin，rend）而不是成员函数:
```cpp
template<typename C, typename V>
void findAndInsert(C& container, // 在容器中查找第⼀次
const V& targetVal, // 出现targetVal的位置，
const V& insertVal) // 然后插⼊insertVal
{
    using std::cbegin; // there
    using std::cend;
    auto it = std::find(cbegin(container), // ⾮成员函数cbegin
                        cend(container), // ⾮成员函数cend
                        targetVal); // c++11 没有添加非成员函数加cbegin，cend，rbegin，rend，crbegin，crend
    container.insert(it, insertVal);
}
// c++11可以通过如下方法实现
template <class C>
auto cbegin(const C& container)->decltype(std::begin(container))
{
    // container 类型为 const C& 
    return std::begin(container); // 对const容器调用⾮成员函数begin（由C++11提供)将产出const_iterator
}
// 如果C是原⽣数组，这个模板也能⼯作。这时，container成为⼀个const数组。C++11为数组提供特化版本的⾮成员函数begin，它返回指向数组第⼀个元素的指针。⼀个const数组的元素也是const，所以对于const数组，⾮成员函数begin返回指向const的指针
```
## Item13-remember
- 优先考虑const_iterator而⾮iterator(C++98需考虑)
- 在最⼤程度通用的代码中，优先考虑⾮成员函数版本的begin，end，rbegin等，而⾮同名成员函数

## 条款 14:如果函数不抛出异常请使用noexcept
在C++11标准化过程中，⼤家⼀致认为异常说明真正有⽤的信息是⼀个函数是否会抛出异常。⾮⿊即⽩，⼀个函数可能抛异常，或者不会。这种"可能-绝不"的⼆元论构成了C++11异常说的基础，从根本上改变了C++98的异常说明。

noexcept可以表示函数不会抛出异常，函数的异常抛出⾏为是客⼾端代码最关⼼的。调⽤者可以查看函数是否声明为noexcept，这个可以影响到调⽤代码的异常安全性和效率。

同时给不抛异常的函数加上noexcept还可以允许编译器⽣成更好的⽬标代码。
```cpp
// 因为在C++98的异常说明中，调⽤栈会展开⾄f的调⽤者，⼀些不合适的动作⽐如程序终⽌也会发⽣。
// 而在  C++11异常说明的运⾏时⾏为明显不同：调⽤栈只是可能在程序终⽌前展开，在⼀个noexcept函数中，当异常传播到函数外，优化器不需要保证运⾏时栈的可展开状态，也不需要保证noexcept函数中的对象按照构造的反序析构
RetType function(params) noexcept; // 极尽所能优化
RetType function(params) throw(); // 较少优化
RetType function(params); // 较少优化
```
移动操作也需要我们考虑使用noexcept

```cpp
std::vector<Widget> vw;
…
Widget w;
… // work with w
vw.push_back(w); // add w to vw
```

当新元素添加到 std::vector ， std::vector 可能没地⽅放它，换句话说， std::vector 的⼤小(size)等于它的容量(capacity)。这时候， std::vector 会分配⼀⽚的新的⼤块内存⽤于存放，然后将元素从已经存在的内存移动到新内存。在C++98中，移动是通过复制⽼内存区的每⼀个元素到新内存区完成的，然后⽼内存区的每个元素发⽣析构。

这种⽅法使得 push_back 可以提供很强的异常安全保证：如果在复制元素期间抛出异常，std::vector 状态保持不变，因为⽼内存元素析构必须建⽴在它们已经成功复制到新内存的前提下。

在C++11中，⼀个很⾃然的优化就是将上述复制操作替换为移动操作。但是很不幸运，这回破坏push_back 的异常安全。如果n个元素已经从⽼内存移动到了新内存区，但异常在移动第n+1个元素时抛出，那么 push_back 操作就不能完成。但是原始的 std::vector 已经被修改：有n个元素已经移动走了。恢复 std::vector ⾄原始状态也不太可能，因为从新内存移动到⽼内存本⾝⼜可能引发异常。

std::vector::push_back 受益于"如果可以就移动，如果必要则复制"策略，并且它不是标准库中唯⼀采取该策略的函数。C++98中还有⼀些函数如 std::vector::reverse , std:;deque::insert 等也受益于这种强异常保证。对于这个函数只有在知晓移动不抛异常的情况下⽤C++11的move替换C++98的copy才是安全的。但是如何知道⼀个函数中的移动操作是否产⽣异常？答案很明显：它检查是否声明noexcept。

swap函数中noexcept也十分重要，标准库的swap是否noexcept有时依赖于⽤⼾定义的swap是否noexcept。⽐如，数组和 std::pair 的swap声明如下：
```cpp
template <class T, size_t N>
void swap(T (&a)[N], // see
          T (&b)[N]) noexcept(noexcept(swap(*a, *b))); // below

template <class T1, class T2>
struct pair {
    …
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
                       noexcept(swap(second, p.second)));
    …
};
```
这些函数视情况noexcept：它们是否noexcept依赖于 noexcept 声明中的表达式是否noexcept。假设有两个Widget数组，不抛异常的交换数组前提是数组中的元素交换不抛异常。对于Widget的交换是否noexcept决定了对于 Widget 数组的交换是否noexcept，反之亦然。类似的，交换两个存放Widget的 std::pair 是否noexcept依赖于Widget的交换是否noexcept。事实上交换⾼层次数据结构是否noexcept取决于它的构成部分的那些低层次数据结构是否异常，这激励你只要可以就提供noexcept的swap函数，因为如果你的函数不提供noexcept保证，其它依赖你的⾼层次swap就不能保证noexcept。

是实际上⼤多数函数都是异常中⽴（exception neutral）的。这些函数⾃⼰不抛异常，但是它们内部的调⽤可能抛出。此时，异常中⽴函数允许那些抛出异常的函数在调⽤链上更进⼀步直到遇到异常处理程序，而不是就地终⽌。异常中⽴函数决不应该声明为noexcept，因为它们可能抛出那种"让它们过吧"的异常，也就是说在当前这个函数内不处理异常，但是⼜不⽴即终⽌程序，而是让调⽤这个函数的函数处理异常。因此⼤多数函数都不应该被指定为noexcept。

**是移动操作和swap——使其不抛异常有重⼤意义，只要可能就应该将它们声明为noexcept；为了noexcept而扭曲函数实现达成⽬的是本末倒置**

在C++98构造函数和析构函数抛出异常是糟糕的代码设计——不管是⽤⼾定义的还是编译器⽣成的构造析构都是noexcept。因此它们不需要声明noexcept。（这么做也不会有问题，只是不合常规）。析构函数⾮隐式noexcept的情况仅当类的数据成员明确声明它的析构函数可能抛出异常（即，声明 noexcept(false) ）。这种析构函数不常⻅，标准库⾥⾯没有。如果⼀个对象的析构函数可能被标准库使⽤，析构函数⼜可能抛异常，那么程序的⾏为是未定义的。

值得注意的是⼀些库接口设计者会区分有宽泛契约(wild contracts)和严格契约(narrow contracts)的函数。有宽泛契约的函数没有前置条件。这种函数不管程序状态如何都能调⽤，它对调⽤者传来的实参不设约束。宽泛契约的函数决不表现出未定义⾏为。反之，没有宽泛契约的函数就有严格契约。对于这些函数，如果违反前置条件，结果将会是未定义的。

假如你在写⼀个参数为std::string的函数f，并且这个函数f很⾃然的决不引发异常。这就在建议我们f应该被声明为noexcept 。现在假如f有⼀个前置条件：类型为std::string的参数的⻓度不能超过32个字符。如果现在调⽤f并传给它⼀个⼤于32字符的参数，函数⾏为将是未定义的，因为违反了 （口头/⽂档）定义的 前置条件，导致了未定义⾏为。f没有义务去检查前置条件，它假设这些前置条件都是满⾜的。（调⽤者有责任确保参数字符不超过32字符等这些假设有效。）。
```cpp
void f(const std::string& s) noexcept; // 前置条件：
                                       // s.length() <= 32
```
如果函数⾥⾯检查前置条件冲突，而f声明了noexcept，这时就会抛出⼀个异常会导致程序终⽌。因为这个原因，区分严格/宽泛契约库设计者⼀般会将noexcept留给宽泛契约函数。

编译器不会为函数实现和异常规范提供⼀致性保障
```cpp
void setup(); // 函数定义另在⼀处
void cleanup();
void doWork() noexcept
{
    setup(); // 前置设置
    … // 真实⼯作
    cleanup(); // 执⾏后置清理
}
```
这⾥，doWork声明为noexcept，即使它调⽤了⾮noexcept函数 setup 和 cleanup 。看起来有点⽭
盾，其实可以猜想 setup 和 cleanup 在⽂档上写明了它们决不抛出异常，即使它们没有写上noexcept。⾄
于为什么明明不抛异常却不写noexcept也是有合理原因的。⽐如，它们可能是⽤C写的库函数的⼀部分。（即使⼀些函数从C标准库移动到了std命名空间，也可能缺少异常规范，std::strlen就是⼀个例⼦，它没有声明
noexcept）。或者它们可能是C++98库的⼀部分，它们不使⽤C++98异常规范的函数的⼀部分，到了C++11还没有修订。

因为有很多合理原因解释为什么noexcept依赖于缺少noexcept保证的函数，所以C++允许这些代码，编译器⼀般也不会给出warnigns。

## Item14-remember
- noexcept是函数接口的⼀部分，这意味着调⽤者会依赖它、
- noexcept函数较之于⾮noexcept函数更容易优化
- noexcept对于移动语义，swap，内存释放函数和析构函数⾮常有⽤
- ⼤多数函数是异常中⽴的，而不是noexcept

