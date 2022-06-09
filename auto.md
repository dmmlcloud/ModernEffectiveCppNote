# auto

## Item5:优先考虑 auto 而⾮显式类型声明

对一个普通变量进行声明，可能会忘记初始化

```cpp
int x = 1;
```

不使用 auto 对⼀个局部变量使⽤解引⽤迭代器的⽅式初始化

```cpp
template<typename It>
void dwim(It b, It e)
{
    // 变量的类型只有编译后知道，这⾥必须使⽤'typename'指定
    while(b!=e){
        typename std::iterator_traits<It>::value_type
        currValue = *b;
    }
}
```

使用 auto

```cpp
int x1; //潜在的未初始化的变量
auto x2; //错误！必须要初始化
auto x3=0; //没问题，x已经定义了
template<typename It>
void dwim(It b,It e)
{
    while(b!=e){
        auto currValue = *b;
        ...
    }
}
```

对于 lambda 表达式也可以用 auto 进行表示

```cpp
auto derefUPLess = [](const std::unique_ptr<Widget> &p1, //专⽤于Widget类型的⽐较函数
const std::unique_ptr<Widget> &p2){return *p1<*p2;};
auto derefUPLess = [](const auto& p1,const auto& p2){return *p1<*p2;}; // c++14 对形参使用auto
```

如果使用 std::function(std::function 是⼀个 C++11 标准模板库中的⼀个模板，它泛化了函数指针的概念。与函数指针只能指向函数不同， std::function 可以指向任何可调⽤对象，也就是那些像函数⼀样能进⾏调⽤的东西。当你声明函数指针时你必须指定函数类型（即函数签名），同样当你创建 std::function 对象时你也需要提供函数签名，由于它是⼀个模板所以你需要在它的模板参数⾥⾯提供)

```cpp
bool(const std::unique_ptr<Widget> &p1, const std::unique_ptr<Widget> &p2);
std::function<bool(const std::unique_ptr<Widget> &p1, const std::unique_ptr<Widget> &p2)> func;
// 如果声明lambda表达式
std::function<bool(const std::unique_ptr<Widget> &p1,
const std::unique_ptr<Widget> &p2)>
dereUPLess = [](const std::unique_ptr<Widget> &p1,
const std::unique_ptr<Widget> &p2){return *p1<*p2;};
```

实例化 std::function 并声明⼀个对象这个对象将会有固定的⼤小。当使⽤这个对象保存⼀个闭包时它
可能⼤小不⾜不能存储，这个时候 std::function 的构造函数。将会在堆上⾯分配内存来存储，这就造成
了使⽤ std::function ⽐ auto 会消耗更多的内存。
使⽤ auto 除了使⽤未初始化的⽆效变量，省略冗⻓的声明类型，直接保存闭包外，它还有⼀个好处是可
以避免⼀个问题，我称之为依赖类型快捷⽅式的问题：

```cpp
std::vector<int> v;
unsigned sz = v.size();
std::vector<int>::size_type st = v.size();
```

一般认为 std::vector`<int>`::size_type 就是 usigned_int，但是对于不同的机器，他们可能不同，在 Windows 64-bit 上 std::vector`<int>`::size_type 是 64 位， unsigned int 是 32 位。此时使用 auto 就不用考虑这样的问题：

```cpp
auto sz =v.size();
```

对于下面的代码，由于 std::unordered_map 的 key 是一个常量，下面 for 循环中应该写成（std::pair<const std::string,int>），否则会根据 m 中的元素创建临时变量，并在结束后删除这些对象

```cpp
std::unordered_map<std::string,int> m;
...
for(const std::pair<std::string,int>& p : m)
{
    ...
}
```

而使用 auto 可以避免这样的问题

```cpp
for(const auto & p : m)
{
    ...
}
```

这两个例⼦说明了显式的指定类型可能会导致你不像看到的类型转换。如果你使⽤ auto 声明⽬标变量你就不必担⼼这个问题。

使⽤ auto 可以帮助我们完成⼀些重构⼯作。举个例⼦，如果⼀个函数返回类型被声明为 int，但是后来你认为将它声明为 long 会更好，调⽤它作为初始化表达式的变量会⾃动改变类型，但是如果你不使⽤ auto 你就不得不在源代码中挨个找到调⽤地点然后修改它们。

## Item5:remember

- auto 变量必须初始化，通常它可以避免⼀些移植性和效率性的问题，也使得重构更⽅便，还能让你少打⼏个字。
- 正如 [Item2](./deducing_types.md) 和 [Item6](#item6如果-auto-不能预期推导出类型使用显示类型初始化) 讨论的，auto 类型的变量可能会踩到⼀些陷阱。

## Item6:如果 auto 不能预期推导出类型，使用显示类型初始化

```cpp
auto highPriority = features(w)[5];
...
processWidget(w,highPriority); //未定义行为！
```

虽然从概念上来说 std::vector 意味着存放 bool，但是 std::vector 的 operator[] 不会返回容器中元素的引用，取而代之它返回⼀个 std::vector::reference 的对象（⼀个嵌套于 std::vector 中的类） 。std::vector::reference 之所以存在是因为 std::vector 指定了它作为代理类。operator[] 返回⼀个代理类来扮演 bool&。要想成功扮演这个角色，bool&适用的上下文 std::vector::reference 也必须⼀样能适用。基于这个特性 std::vector::reference 可以隐式的转化为 bool。
所以在当定义 highPriority 使用 bool 时，

```cpp
bool highPriority = features(w)[5]; //显式的声明highPriority的类型
// features(w)[5]返回std::vector<bool>::reference，然后强制类型转换成bool
```

而在使用 auto 时，auto 推导 highPriority 的类型为 std::vector::reference ，但是 highPriority 对象没有第五 bit 的值。这个值取决于 std::vector::reference 的具体实现。其中的⼀种实现是这样的（ std::vector::reference ）对象包含⼀个指向 word 的指针，然后加上方括号中的偏移实现被引用 bit 这样的行为。然后再来考虑 highPriority 初始化表达的意思，注意这里假设 std::vector::reference 就是刚提到的实现方式。
调用 feature 将返回⼀个 std::vector，这个对象没有名字，为了方便我们的讨论，我这里叫他 temp，operator[] 被 temp 调用，然后然后的 std::vector::reference 包含⼀个指针，这个指针指向⼀个 temp 里面的 word，加上相应的偏移,。highPriority 是⼀个 std::vector::reference 的拷贝，所以 highPriority 也包含⼀个指针，指向 temp 中的⼀个 word，加上合适的偏移，这里是 5.在这个语句解释的时候 temp 将会被销毁，因为它是⼀个临时变量。因此 highPriority 包含一个悬置的指针，如果用于 processWidget 调用中将会造成未定义行为：

```cpp
processWidget(w,highPriority); //未定义行为！
//highPriority包含⼀个悬置指针
```

类似的还有 Matrix 在为了计算方便时，operator+重载返回一个 Sum<Matrix, Matrix>类型的对象。

**所以作为一个通则，不可见的代理类通常不适用于 auto。这样类型的对象的生命期通常不会设计为能活过一条语句。**

如果想在不知道有没有使用代理类的情况下使用 auto，解决方案是强制使用⼀个不同的类型推导形式，这种方法我通常称之为显式类型初始器惯用法。即：使用 auto 声明一个变量，然后对表达式强制类型转换得出你期望的推导结果。

```cpp
auto highPriority = static_cast(features(w)[5])

auto sum = static_cast(m1+m2+m3+m4);
```

同时这种方法也可以用于强调你声明了一个变量类型，它的类型不同于初始化表达式的类型。

```cpp
double calEpsilon();
float ep = calEpsilon();
auto ep = static_cast<float>(calEpsilon());

auto index = static_cast(d * size()); // 计算下标时，d为double类型
```

## Item6:remember

- 不可见的代理类可能会使 auto 从表达式中推导出“错误的”类型
- 显式类型初始器惯用法强制 auto 推导出你想要的结果
