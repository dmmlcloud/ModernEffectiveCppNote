# Rvalue References, Move Semantics, and Perfect Forwarding

- 移动语义使编译器有可能用廉价的移动操作来代替昂贵的拷贝操作。正如拷贝构造函数和拷贝赋值操作符给了你控制拷贝语义的权力，移动构造函数和移动赋值操作符也给了你控制移动语义的权力。移动语义也允许创建只可移动（move-only）的类型，例如 std::unique_ptr，std::future 和 std::thread。

- 完美转发使接收任意数量实参的函数模板成为可能，它可以将实参转发到其他的函数，使目标函数接收到的实参与被传递给转发函数的实参保持一致。

但 std::move 并不移动任何东西，完美转发也并不完美。移动操作并不永远比复制操作更廉价；即便如此，它也并不总是像你期望的那么廉价。而且，它也并不总是被调用，即使在当移动操作可用的时候。构造“type&&”也并非总是代表一个右值引用。

在本章的这些小节中，非常重要的一点是要牢记形参永远是左值，即使它的类型是一个右值引用。比如，假设

```cpp
void f(Widget&& w);
```

形参 w 在这个函数中是一个左值，即使它的类型是一个 rvalue-reference-to-Widget

## Item23:理解 std::move 和 std::forward

为了了解 std::move 和 std::forward，一种有用的方式是从 **它们不做什么这个角度来了解它们。std::move 不移动（move）任何东西，std::forward 也不转发（forward）任何东西。** 在运行时，它们不做任何事情。它们不产生任何可执行代码，一字节也没有。

它们只是执行转换操作，std::move 无条件的将它的实参转换为右值，而 std::forward 只在特定情况满足时下进行转换。以下是一个 std::move 的示例实现：

```cpp
template<typename T>                            //在std命名空间
typename remove_reference<T>::type&&
move(T&& param)
{
    using ReturnType =                          //别名声明，见条款9
        typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}

// c++14
template<typename T>                            //在std命名空间
decltype(auto) move(T&& param)
{
    using ReturnType =                          //别名声明，见条款9
        typename remove_reference<T>::type&&;

    return static_cast<ReturnType>(param);
}
```

std::move 接受一个对象的引用（准确的说，一个通用引用（universal reference），见[Item24](#item24))，返回一个指向同对象的引用。

该函数返回类型的&&部分表明 std::move 函数返回的是一个右值引用，但是，正如 [Item28](#item28) 所解释的那样，如果类型 T 恰好是一个左值引用，那么 T&&将会成为一个左值引用。为了避免如此，type trait（见 [traits](traits.md)）std::remove_reference 应用到了类型 T 上，因此确保了&&被正确的应用到了一个不是引用的类型上。这保证了 std::move 返回的真的是右值引用，这很重要，因为函数返回的右值引用是右值。因此，std::move 将它的实参转换为一个右值，这就是它的全部作用。**它只进行转换，不移动任何东西。**

std::move 并不能保证移动操作真正被执行，对于一个类，根据[Item41](tweaks.md)，声明一个值传递的形参：：

```cpp
class Annotation {
public:
    explicit Annotation(const std::string text)  //将会被复制的形参，如同条款41所说，值传递，因为只需要读取设置为const
    ：value(std::move(text))                     // 避免一次复制操作的代价
    {...}
};
private:
    std::string value;
```

这段代码可以编译，可以链接，可以运行。这段代码将数据成员 value 设置为 text 的值。这段代码与你期望中的完美实现的唯一区别，是 text 并不是被移动到 value，而是被拷贝。诚然，text 通过 std::move 被转换到右值，但是 text 被声明为 const std::string，所以在转换之前，text 是一个左值的 const std::string，而转换的结果是一个右值的 const std::string，但是纵观全程，const 属性一直保留。

当编译器决定哪一个 std::string 的构造函数被调用时，考虑它的作用，将会有两种可能性：

```cpp
class string {                  //std::string事实上是
public:                         //std::basic_string<char>的类型别名
    …
    string(const string& rhs);  //拷贝构造函数
    string(string&& rhs);       //移动构造函数
    …
};
```

std::move(text)的结果是一个 const std::string 的右值。这个右值不能被传递给 std::string 的移动构造函数，因为**移动构造函数只接受一个指向 non-const 的 std::string 的右值引用**。然而，**该右值却可以被传递给 std::string 的拷贝构造函数，因为 lvalue-reference-to-const 允许被绑定到一个 const 右值上**。因此，std::string 在成员初始化的过程中调用了拷贝构造函数，即使 text 已经被转换成了右值。这样是为了确保维持 const 属性的正确性。从一个对象中移动出某个值通常代表着修改该对象，所以语言不允许 const 对象被传递给可以修改他们的函数（例如移动构造函数）。

可以总结出两点。第一，不要在你希望能移动对象的时候，声明他们为 const。对 const 对象的移动请求会悄无声息的被转化为拷贝操作。第二点，std::move 不仅不移动任何东西，而且它也不保证它执行转换的对象可以被移动。关于 std::move，你能确保的唯一一件事就是**将它应用到一个对象上，你能够得到一个右值。**

std::forward 是有条件的转换，在一定条件下才执行转换，常见的情景是一个模板函数，接收一个通用引用形参，并将它传递给另外的函数：

```cpp
void process(const Widget& lvalArg);        //处理左值
void process(Widget&& rvalArg);             //处理右值

template<typename T>                        //用以转发param到process的模板
void logAndProcess(T&& param)
{
    auto now =                              //获取现在时间
        std::chrono::system_clock::now();

    makeLogEntry("Calling 'process'", now);
    process(std::forward<T>(param));
}

Widget w;

logAndProcess(w);               //用左值调用
logAndProcess(std::move(w));    //用右值调用
```

我们在调用的时候，希望如果传入的是左值就按左值处理，右值就按右值处理，但是在函数中 param，正如所有的其他函数形参一样，是一个左值，每次在函数 logAndProcess 内部对函数 process 的调用，都会因此调用函数 process 的左值重载版本。为防如此，我们需要一种机制：当且仅当传递给函数 logAndProcess 的用以初始化 param 的实参是一个右值时，param 会被转换为一个右值。这就是 std::forward 做的事情。这就是为什么 std::forward 是一个有条件的转换：**它的实参用右值初始化时，转换为一个右值。**

std::forward 可以通过模板参数中 T 分辨 param 是被左值初始化的还是右值初始化的（见[Item1](deducing_types.md)和[Item28](#item28)）

std::forward 有时可以替代 std::move：

```cpp
class Widget {
public:
    Widget(Widget&& rhs)
    : s(std::move(rhs.s))
    { ++moveCtorCalls; }

    …

private:
    static std::size_t moveCtorCalls;
    std::string s;
};

class Widget{
public:
    Widget(Widget&& rhs)                    //不自然，不合理的实现
    : s(std::forward<std::string>(rhs.s))
    { ++moveCtorCalls; }

    …

}
```

但是：

- std::move 只需要一个函数实参（rhs.s），而 std::forward 不但需要一个函数实参（rhs.s），还需要一个模板类型实参 std::string
- 我们传递给 std::forward 的类型应当是一个 non-reference，因为惯例是传递的实参应该是一个右值([Item28](#item28))

更重要的是，std::move 的使用代表着无条件向右值的转换，而使用 std::forward 只对绑定了右值的引用进行到右值转换。这是两种完全不同的动作。前者是典型地为了移动操作，而后者只是传递（亦为转发）一个对象到另外一个函数，保留它原有的左值属性或右值属性。

## Item23-remember

- std::move 执行到右值的无条件的转换，但就自身而言，它不移动任何东西。
- std::forward 只有当它的参数被绑定到一个右值时，才将参数转换为右值。
- std::move 和 std::forward 在运行期什么也不做。

## Item24:区分通用引用与右值引用

T&&不一定是右值引用

```cpp
void f(Widget&& param);             //右值引用
Widget&& var1 = Widget();           //右值引用
auto&& var2 = var1;                 //不是右值引用

template<typename T>
void f(std::vector<T>&& param);     //右值引用

template<typename T>
void f(T&& param);                  //不是右值引用
```

T&&既可以是右值引用，也可以是左值引用。这种引用在源码里看起来像右值引用（即“T&&”），但是它们可以表现得像是左值引用（即“T&”）。它们的二重性使它们既可以绑定到右值上（就像右值引用），也可以绑定到左值上（就像左值引用）。 此外，它们还可以绑定到 const 或者 non-const 的对象上，也可以绑定到 volatile 或者 non-volatile 的对象上，甚至可以绑定到既 const 又 volatile 的对象上。它们可以绑定到几乎任何东西，叫做通用引用（universal references）。他也可以应用到 std::forward 上[Item25](#item25)
在两种情况下会出现通用引用：

- 函数模板形参
  ```cpp
  template<typename T>
  void f(T&& param);                  //param是一个通用引用
  ```
- auto 声明
  ```cpp
  auto&& var2 = var1;                 // var2是一个通用引用
  ```
  这两种情况的共同之处就是都存在**类型推导**（type deduction）。在模板 f 的内部，param 的类型需要被推导，而在变量 var2 的声明中，var2 的类型也需要被推导。

同以下的例子相比较（同样来自于上面的示例代码），下面的例子不带有类型推导。如果你看见“T&&”不带有类型推导，那么你看到的就是一个右值引用：

```cpp
void f(Widget&& param);         //没有类型推导，
                                //param是一个右值引用
Widget&& var1 = Widget();       //没有类型推导，
                                //var1是一个右值引用

```

因为通用引用是引用，所以它们必须被初始化。一个通用引用的初始值决定了它是代表了右值引用还是左值引用。如果初始值是一个右值，那么通用引用就会是对应的右值引用，如果初始值是一个左值，那么通用引用就会是一个左值引用。对那些是函数形参的通用引用来说，初始值在调用函数的时候被提供：

```cpp
template<typename T>
void f(T&& param);              //param是一个通用引用

Widget w;
f(w);                           //传递给函数f一个左值；param的类型
                                //将会是Widget&，也即左值引用

f(std::move(w));                //传递给f一个右值；param的类型会是
                                //Widget&&，即右值引用
```

而正如第一个例子中那样，T&&是通用引用，vector<T>&&是右值引用，在传递左值时也会报错：

```cpp
std::vector<int> v;
f(v);                           //错误！不能将左值绑定到右值引用
```

对通用引用不能使用 const 对其进行限定，这会使它变为右值引用：

```cpp
template <typename T>
void f(const T&& param);        //param是一个右值引用
```

由于在模板内部并不保证一定会发生类型推导，所以 T&&不一定是通用引用：

```cpp
template<class T, class Allocator = allocator<T>>   //来自C++标准
class vector
{
public:
    void push_back(T&& x);
    …
}
```

这里并没有发生类型推导，因为 push_back 在有一个特定的 vector 实例之前不可能存在，而实例化 vector 时的类型已经决定了 push_back 的声明。

```cpp
std::vector<Widget> v;
// vector会被实例化成：
class vector<Widget, allocator<Widget>> {
public:
    void push_back(Widget&& x);             //右值引用
    …
};
```

而使用 emplace_back，却包含类型推导：

```cpp
template<class T, class Allocator = allocator<T>>   //依旧来自C++标准
class vector {
public:
    template <class... Args>
    void emplace_back(Args&&... args);
    …
};
```

在 emplace_back 中类型参数（type parameter）Args 是独立于 vector 的类型参数 T 的，所以 Args 会在每次 emplace_back 被调用的时候被推导。（Args 实际上是一个 parameter pack，而不是一个类型参数，但是为了方便讨论，我们可以把它当作是一个类型参数。）

类型声明为 auto&&的变量是通用引用，因为会发生类型推导，并且它们具有正确形式(T&&)。auto 类型的通用引用不如函数模板形参中的通用引用常见，但是它们在 C++11 中常常突然出现。而它们在 C++14 中出现得更多，因为 C++14 的 lambda 表达式可以声明 auto&&类型的形参。举个例子，如果你想写一个 C++14 标准的 lambda 表达式，来记录任意函数调用的时间开销，你可以这样写：

```cpp
auto timeFuncInvocation =
    [](auto&& func, auto&&... params)           //C++14
    {
        start timer;
        std::forward<decltype(func)>(func)(     //对params调用func
            std::forward<delctype(params)>(params)...
        );
        stop timer and record elapsed time;
    };
```

func 是一个通用引用，可以被绑定到任何可调用对象，无论左值还是右值。args 是 0 个或者多个通用引用（即它是个通用引用 parameter pack），它可以绑定到任意数目、任意类型的对象上。多亏了 auto 类型的通用引用，函数 timeFuncInvocation 可以对近乎任意（见[Item30](./lambda.md)）函数进行计时。

通用引用的基础是**引用折叠**（[Item28](#item28)）

## Item24-remember

- 如果一个函数模板形参的类型为 T&&，并且 T 需要被推导得知，或者如果一个对象被声明为 auto&&，这个形参或者对象就是一个通用引用。
- 如果类型声明的形式不是标准的 type&&，或者如果类型推导没有发生，那么 type&&代表一个右值引用。
- 通用引用，如果它被右值初始化，就会对应地成为右值引用；如果它被左值初始化，就会成为左值引用。

## Item25:对右值引用使用 std::move，对通用引用使用 std::forward

如果函数有个右值引用形参，意味着希望传递这样的对象给其他函数，允许那些函数利用对象的右值性，可以使用 std::move 将形参转化成右值：

```cpp
class Widget {
public:
    Widget(Widget&& rhs)        //rhs是右值引用
    : name(std::move(rhs.name)),
      p(std::move(rhs.p))
      { … }
    …
private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};
```

对于通用引用形参，可以使用 std::forward 进行操作：

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)           //newName是通用引用
    { name = std::forward<T>(newName); }

    …
};
```

总而言之，**当把右值引用转发给其他函数时，右值引用应该被无条件转换为右值（通过 std::move），因为它们总是绑定到右值；当转发通用引用时，通用引用应该有条件地转换为右值（通过 std::forward），因为它们只是有时绑定到右值。**

根据[Item23](#item23理解-stdmove-和-stdforward)，可以在右值引用上使用 std::forward 表现出适当的行为，但是代码较长，容易出错，所以应该避免在右值引用上使用 std::forward。更糟的是在通用引用上使用 std::move，这可能会意外改变左值（比如局部变量）：

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName)       //通用引用可以编译，
    { name = std::move(newName); }  //但是代码太太太差了！
    …

private:
    std::string name;
    std::shared_ptr<SomeDataStructure> p;
};

std::string getWidgetName();        //工厂函数

Widget w;

auto n = getWidgetName();           //n是局部变量

w.setName(n);                       //把n移动进w！

…                                   //现在n的值未知
```

局部变量 n 被传递给 w.setName，调用方可能认为这是对 n 的只读操作，但是因为 setName 内部使用 std::move 无条件将传递的引用形参转换为右值，n 的值被移动进 w.name，调用 setName 返回时 n 最终变为未定义的值。

虽然对于只读操作应该限定形参为 const，而通用引用不能使用 const，所以应该分别对左值右值情况分别进行重载：

```cpp
class Widget {
public:
    void setName(const std::string& newName)    //用const左值设置
    { name = newName; }

    void setName(std::string&& newName)         //用右值设置
    { name = std::move(newName); }

    …
};
```

但是这样也有缺点：首先编写和维护的代码更多（两个函数而不是单个模板）；其次，效率下降。比如，考虑如下场景：

```cpp
w.setName("Adela Novak");
```

使用通用引用的版本的 setName，字面字符串“Adela Novak”可以被传递给 setName，再传给 w 内部 std::string 的赋值运算符。w 的 name 的数据成员通过字面字符串直接赋值，没有临时 std::string 对象被创建。但是，setName 重载版本，会有一个临时 std::string 对象被创建，setName 形参绑定到这个对象，然后这个临时 std::string 移动到 w 的数据成员中。一次 setName 的调用会包括 std::string 构造函数调用（创建中间对象），std::string 赋值运算符调用（移动 newName 到 w.name），std::string 析构函数调用（析构中间对象）。这比调用接受 const char\*指针的 std::string 赋值运算符开销昂贵许多。增加的开销根据实现不同而不同，这些开销是否值得担心也跟应用和库的不同而有所不同，但是事实上，将通用引用模板替换成对左值引用和右值引用的一对函数重载在某些情况下会导致运行时的开销。如果把例子泛化，Widget 数据成员是任意类型（而不是知道是个 std::string），性能差距可能会变得更大，因为不是所有类型的移动操作都像 std::string 开销较小（参看[Item29](#item29)）。

但最重要的还是设计的可扩展性差，Widget::setName 有一个形参，因此需要两种重载实现，但是对于有更多形参的函数，每个都可能是左值或右值，重载函数的数量几何式增长：n 个参数的话，就要实现 2n 种重载。这还不是最坏的。有的函数——实际上是函数模板——接受无限制个数的参数，每个参数都可以是左值或者右值。此类函数的典型代表是 std::make_shared，还有对于 C++14 的 std::make_unique（见 Item21）。查看他们的的重载声明：

```cpp
template<class T, class... Args>                //来自C++11标准
shared_ptr<T> make_shared(Args&&... args);

template<class T, class... Args>                //来自C++14标准
unique_ptr<T> make_unique(Args&&... args);
```

对于这种函数，对于左值和右值分别重载就不能考虑了：通用引用是仅有的实现方案。
如果需要在一个函数中多次使用绑定到右值引用或者通用引用的对象，并且确保在完成其他操作前，这个对象不会被移动。我们需要只在最后一次操作中使用 std::move or std::forward:

```cpp
template<typename T>
void setSignText(T&& text)                  //text是通用引用
{
  sign.setText(text);                       //使用text但是不改变它

  auto now =
      std::chrono::system_clock::now();     //获取现在的时间

  signHistory.add(now,
                  std::forward<T>(text));   //有条件的转换为右值
}
```

在有些稀少的情况下，你需要调用 std::move_if_noexcept 代替 std::move。（参考[Item14](./moving2modern_cpp.md)，例如 std::vector 的 push_back，确定该类型的移动构造是否是 noexcept，视情况使用左值或右值）。

如果你在按值返回的函数中，返回值绑定到右值引用或者通用引用上，需要对返回的引用使用 std::move 或者 std::forward。要了解原因，考虑两个矩阵相加的 operator+函数，左侧的矩阵为右值（可以被用来保存求值之后的和）：

```cpp
Matrix                              //按值返回
operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return std::move(lhs);	        //移动lhs到返回值中
}
// 如果不使用std::move
Matrix                              //同之前一样
operator+(Matrix&& lhs, const Matrix& rhs)
{
    lhs += rhs;
    return lhs;                     //拷贝lhs到返回值中
}
```

如果不将其转化为右值，会强制编译器拷贝它到返回值的内存空间。假定 Matrix 支持移动操作，并且比拷贝操作效率更高，在 return 语句中使用 std::move 的代码效率更高。而如果不支持移动操作，他会使用拷贝操作不会影响结果和性能。

使用通用引用和 std::forward 的情况类似。考虑函数模板 reduceAndCopy 收到一个未规约（unreduced）对象 Fraction，将其规约，并返回一个规约后的副本。如果原始对象是右值，可以将其移动到返回值中（避免拷贝开销），但是如果原始对象是左值，必须创建副本，因此如下代码：

```cpp
template<typename T>
Fraction                            //按值返回
reduceAndCopy(T&& frac)             //通用引用的形参
{
    frac.reduce();
    return std::forward<T>(frac);		//移动右值，或拷贝左值到返回值中，如果std::forward被忽略，frac就被无条件复制到reduceAndCopy的返回值内存空间。
}
```

如果将该方法应用到普通函数中进行优化：

```cpp
Widget makeWidget()                 //makeWidget的“拷贝”版本
{
    Widget w;                       //局部对象
    …                               //配置w
    return w;                       //“拷贝”w到返回值中
}
// to
Widget makeWidget()                 //makeWidget的移动版本
{
    Widget w;
    …
    return std::move(w);            //移动w到返回值中（不要这样做！）
}
```

而由于编译器的优化，makeWidget 的“拷贝”版本可以避免复制局部变量 w 的需要，**通过在分配给函数返回值的内存中构造 w 来实现**。这就是所谓的返回值优化（return value optimization，RVO），这在 C++标准中已经实现了。

编译器可能会在按值返回的函数中消除对局部对象的拷贝（或者移动），如果满足：

- （1）局部对象与函数返回值的类型相同；
- （2）局部对象就是要返回的东西。（适合的局部对象包括大多数局部变量（比如 makeWidget 里的 w），还有作为 return 语句的一部分而创建的临时对象。函数形参不满足要求。一些人将 RVO 的应用区分为命名的和未命名的（即临时的）局部对象，限制了 RVO 术语应用到未命名对象上，并把对命名对象的应用称为命名返回值优化（named return value optimization，NRVO）。）
  而上述例子满足这两点。

而如果使用 std::move，会将 w 的内容移动到 makeWidget 的返回值位置，因为 std::move(w)不再是局部对象，而是局部对象的引用，不满足 RVO 的条件。

但该优化不一定会实现，比如当函数不同控制路径返回不同局部变量时。如果这样，你可能会愿意以移动的代价来保证不会产生拷贝。那就是，极可能仍然认为应用 std::move 到一个要返回的局部对象上是合理的，只因为可以不再担心拷贝的代价。

但那种情况下，应用 std::move 到一个局部对象上仍然是一个坏主意。C++标准关于 RVO 的部分表明，**如果满足 RVO 的条件，但是编译器选择不执行拷贝消除，则返回的对象必须被视为右值**。实际上，标准要求当 RVO 被允许时，或者实行拷贝消除，或者将 std::move 隐式应用于返回的局部对象。因此，在 makeWidget 的“拷贝”版本中，要么通过 RVO 进行了优化，要么将对象变成了右值（不需要额外的 std::move）

对于形参而言，他们虽然不能通过 RVO 优化，但是当他们作为返回值的时候，会被视为右值：

```cpp
Widget makeWidget(Widget w)         //传值形参，与函数返回的类型相同
{
    …
    return w;
}
// 被编译器视为
Widget makeWidget(Widget w)
{
    …
    return std::move(w);
}
```

这意味着，如果对从按值返回的函数返回来的局部对象使用 std::move，你并不能帮助编译器（如果不能实行拷贝消除的话，他们必须把局部对象看做右值），而是阻碍其执行优化选项。

## Item25-remember

- 最后一次使用时，在右值引用上使用 std::move，在通用引用上使用 std::forward。
- 对按值返回的函数要返回的右值引用和通用引用，执行相同的操作。
- 如果局部对象可以被返回值优化消除，就绝不使用 std::move 或者 std::forward。

## Item26: 避免在通用引用上重载

假定你需要写一个函数，它使用名字作为形参，打印当前日期和时间到日志中，然后将名字加入到一个全局数据结构中：

```cpp
std::multiset<std::string> names;           //全局数据结构
void logAndAdd(const std::string& name)
{
    auto now =                              //获取当前时间
        std::chrono::system_clock::now();
    log(now, "logAndAdd");                  //志记信息
    names.emplace(name);                    //把name加到全局数据结构中；
}                                           //emplace的信息见条款42
std::string petName("Darla");
logAndAdd(petName);                     //传递左值std::string
logAndAdd(std::string("Persephone"));	//传递右值std::string
logAndAdd("Patty Dog");                 //传递字符串字面值
```

在第一个调用中，logAndAdd 的形参 name 绑定到变量 petName。在 logAndAdd 中 name 最终传给 names.emplace。因为 name 是左值，会拷贝到 names 中。没有方法避免拷贝，因为是左值（petName）传递给 logAndAdd 的。

在第二个调用中，形参 name 绑定到右值（显式从“Persephone”创建的临时 std::string）。name 本身是个左值，所以它被拷贝到 names 中，但是我们意识到，原则上，它的值可以被移动到 names 中。本次调用中，我们有个拷贝代价，但是我们应该能用移动勉强应付。

在第三个调用中，形参 name 也绑定一个右值，但是这次是通过“Patty Dog”隐式创建的临时 std::string 变量。就像第二个调用中，name 被拷贝到 names，但是这里，传递给 logAndAdd 的实参是一个字符串字面量。如果直接将字符串字面量传递给 emplace，就不会创建 std::string 的临时变量，而是直接在 std::multiset 中通过字面量构建 std::string。在第三个调用中，我们有个 std::string 拷贝开销，但是我们连移动开销都不想要，更别说拷贝的。

可以通过通用引用提高效率：

```cpp
template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name)); // 根据Item25使用std::forward转发
}

std::string petName("Darla");           //跟之前一样
logAndAdd(petName);                     //跟之前一样，拷贝左值到multiset
logAndAdd(std::string("Persephone"));	//移动右值而不是拷贝它
logAndAdd("Patty Dog");                 //在multiset直接创建std::string
                                        //而不是拷贝一个临时std::string
```

但是对于其他类型的操作，我们仍需要重载：

```cpp
std::string nameFromIdx(int idx);   //返回idx对应的名字

void logAndAdd(int idx)             //新的重载
{
    auto now = std::chrono::system_lock::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}
std::string petName("Darla");           //跟之前一样

logAndAdd(petName);                     //跟之前一样，
logAndAdd(std::string("Persephone")); 	//这些调用都去调用
logAndAdd("Patty Dog");                 //T&&重载版本

logAndAdd(22);                          //调用int重载版本

// 但如果使用其他类型的调用
short nameIdx;
…                                       //给nameIdx一个值
logAndAdd(nameIdx);                     //错误！
```

有两个重载的 logAndAdd。使用通用引用的那个推导出 T 的类型是 short，因此可以精确匹配。对于 int 类型参数的重载也可以在 short 类型提升后匹配成功。根据正常的重载解决规则，精确匹配优先于类型提升的匹配，所以被调用的是通用引用的重载。

使用通用引用的函数在 C++中是最贪婪的函数。它们几乎可以精确匹配任何类型的实参（极少不适用的实参在[Item30](#item30)中介绍）。这也是把重载和通用引用组合在一块是糟糕主意的原因：通用引用的实现会匹配比开发者预期要多得多的实参类型。

完美转发构造函数也会导致很多问题，就像在上面的例子中，传递一个不是 int 的整型变量（比如 std::size_t，short，long 等）会调用通用引用的构造函数而不是 int 的构造函数，这会导致编译错误。这里这个问题甚至更糟糕，因为 Person 中存在的重载比肉眼看到的更多。在[Item17](./moving2modern_cpp.md)中说明，在适当的条件下，C++会生成拷贝和移动构造函数，即使类包含了模板化的构造函数，模板函数能实例化产生与拷贝和移动构造函数一样的签名，也在合适的条件范围内。如果拷贝和移动构造被生成，Person 类看起来就像这样：

```cpp
class Person {
public:
    template<typename T>            //完美转发的构造函数
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}

    explicit Person(int idx);       //int的构造函数

    Person(const Person& rhs);      //拷贝构造函数（编译器生成）
    Person(Person&& rhs);           //移动构造函数（编译器生成）
    …
};

Person p("Nancy");
auto cloneOfP(p);                   //从p创建新Person；这通不过编译！
```

这里我们试图通过一个 Person 实例创建另一个 Person，显然应该调用拷贝构造即可。但是这份代码不是调用拷贝构造函数，而是调用完美转发构造函数。然后，完美转发的函数将尝试使用 Person 对象 p 初始化 Person 的 std::string 数据成员，编译器就会报错。

编译器的理由如下：cloneOfP 被 non-const 左值 p 初始化，这意味着模板化构造函数可被实例化为采用 Person 类型的 non-const 左值。实例化之后，Person 类看起来是这样的：

```cpp
class Person {
public:
    explicit Person(Person& n)          //由完美转发模板初始化
    : name(std::forward<Person&>(n)) {}

    explicit Person(int idx);           //同之前一样

    Person(const Person& rhs);          //拷贝构造函数（编译器生成的）
    …
};
```

其中 p 被传递给拷贝构造函数或者完美转发构造函数。调用拷贝构造函数要求在 p 前加上 const 的约束来满足函数形参的类型，而调用完美转发构造不需要加这些东西。从模板产生的重载函数是更好的匹配，所以编译器按照规则：调用最佳匹配的函数。“拷贝”non-const 左值类型的 Person 交由完美转发构造函数处理，而不是拷贝构造函数。

## Item26-remember

## Item27:

## Item27-remembe

## Item28:

## Item28-rememberr

## Item29:

## Item29-remember

## Item30:

## Item30-remember

```

```
