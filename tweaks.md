# 微调

对于 C++中的通用技术和特性，总是存在适用和不适用的场景。除了本章覆盖的两个例外，描述什么场景使用哪种通用技术通常来说很容易。这两个例外是传值（pass by value）和安置（emplacement）。决定何时使用这两种技术受到多种因素的影响，本书提供的最佳建议是在使用它们的同时仔细考虑清楚，尽管它们都是高效的现代 C++编程的重要角色。接下来的条款提供了使用它们来编写软件是否合适的所需信息。

## Item 41:对于那些可移动总是被拷贝的形参使用传值方式 由 @wendajiang 贡献

有些函数的形参是可拷贝的。（在本条款中，“拷贝”一个形参通常意思是，将形参作为拷贝或移动操作的源对象。简介中提到，C++没有术语来区分拷贝操作得到的副本和移动操作得到的副本。）比如说，addName 成员函数可以拷贝自己的形参到一个私有容器。为了提高效率，应该拷贝左值，移动右值。

```cpp
class Widget {
public:
    void addName(const std::string& newName)    //接受左值；拷贝它
    { names.push_back(newName); }

    void addName(std::string&& newName)         //接受右值；移动它；std::move
    { names.push_back(std::move(newName)); }    //的使用见条款25
    …
private:
    std::vector<std::string> names;
};
```

这是可行的，但是需要编写两个函数来做本质相同的事。这有点让人难受：两个函数声明，两个函数实现，两个函数写说明，两个函数的维护。唉。

此外，目标代码中会有两个函数——你可能会担心程序的空间占用。这种情况下，两个函数都可能内联，可能会避免同时两个函数同时存在导致的代码膨胀问题，但是一旦没有被内联，目标代码就会出现两个函数。

另一种方法是使 addName 函数成为具有通用引用的函数模板（参考 [Item24](./rvalue_references%26more.md)）：

```cpp
template <typename T>
void addName(T&& newName) {
    names.push_back(std::forward<T>(newName));
    ...
}
```

这减少了源代码的维护工作，但是通用引用会导致其他复杂性。作为模板，addName 的实现必须放置在头文件中。在编译器展开的时候，可能会产生多个函数，因为不止为左值和右值实例化，也可能为 std::string 和可转换为 std::string 的类型分别实例化为多个函数（参考 [Item25](./rvalue_references%26more.md)）。同时有些实参类型不能通过通用引用传递（参考 [Item30](./rvalue_references%26more.md)），而且如果传递了不合法的实参类型，编译器错误会令人生畏（参考 [Item27](./rvalue_references%26more.md)）。

一种使 addName 可以左值拷贝，右值移动，只用处理一个函数（源代码和目标代码中）得方法就是，放弃一般 c++编程的规则：避免在传递用户定义的对象时使用传值方式。使用按值传递，这在 addName 函数中是一种完全合理的策略。

在讨论为什么对于 addName 中的 newName 按值传递非常合理之前，先考虑如何实现：

```cpp
class Widget {
public:
    void addName(std::string newName) {         //接受左值或右值；移动它
        names.push_back(std::move(newName));
    }
    …
}
```

该代码唯一可能令人困惑的部分就是对形参 newName 使用 std::move。std::move 典型的应用场景是用在右值引用，但是在这里：

- newName 是与调用者传进来的对象相完全独立的一个对象，所以改变 newName 不会影响到调用者；
- 这就是 newName 的最后一次使用，所以移动它不会影响函数内此后的其他代码。

只有一个 addName 函数的事实，解释了在源代码和目标代码中，我们怎样避免了代码重复。我们没有使用通用引用，不会导致头文件膨胀、奇怪的失败情况、或者令人困惑的编译错误消息，但是这种设计不免让人顾虑按值传递的销量。

在 C++98 中，可以肯定确实开销会大。无论调用者传入什么，形参 newName 都是由拷贝构造出来。但是在 C++11 中，只有在左值实参情况下，addName 被拷贝构造出来；对于右值，它会被移动构造：

```cpp
Widget w;
…
std::string name("Bart");
w.addName(name);            //使用左值调用addName
…
w.addName(name + "Jenne");  //使用右值调用addName（见下）
```

第一处调用 addName（当 name 被传递时），形参 newName 是使用左值被初始化。newName 因此是被拷贝构造，就像在 C++98 中一样。
第二处调用，newName 使用 std::string 对象被初始化，这个 std::string 对象是调用 std::string 的 operator+（即 append 操作）得到的结果。这个对象是一个右值，因此 newName 是被移动构造的。

对上述三种写法的开销进行分析：

```cpp
class Widget {                                  //方法1：对左值和右值重载
public:
    void addName(const std::string& newName)
    { names.push_back(newName); } // rvalues
    void addName(std::string&& newName)
    { names.push_back(std::move(newName)); }
    …
private:
    std::vector<std::string> names;
};

class Widget {                                  //方法2：使用通用引用
public:
    template<typename T>
    void addName(T&& newName)
    { names.push_back(std::forward<T>(newName)); }
    …
};

class Widget {                                  //方法3：传值
public:
    void addName(std::string newName)
    { names.push_back(std::move(newName)); }
    …
};

Widget w;
…
std::string name("Bart");
w.addName(name);                                //传左值
…
w.addName(name + "Jenne");                      //传右值
```

- **重载**：无论传递左值还是传递右值，调用都会绑定到一个叫 newName 的引用上。从拷贝和移动操作方面看，这个过程零开销。左值重载中，newName 拷贝到 Widget::names 中，右值重载中，移动进去。**开销总结：左值一次拷贝，右值一次移动**。
- **使用通用引用**：同重载一样，调用也绑定到 addName 这个引用上，没有开销。由于使用了 std::forward，左值 std::string 实参会拷贝到 Widget::names，右值 std::string 实参移动进去。对 std::string 实参来说，**开销同重载方式一样：左值一次拷贝，右值一次移动**。
- **按值传递**：无论传递左值还是右值，都必须构造 newName 形参。如果传递的是左值，需要拷贝的开销，如果传递的是右值，需要移动的开销。在函数的实现中，newName 总是采用移动的方式到 Widget::names。**开销总结：左值实参，一次拷贝一次移动，右值实参两次移动**。对比按引用传递的方法，对于左值或者右值，均多出一次移动操作。

回顾 Item 的内容：**对于移动成本低且总是被拷贝的可拷贝形参，考虑按值传递**

主要基于以下四点：

1. 应该仅**考虑**使用传值方式（并非一定使用）。虽然，该方式只需要编写一个函数，只会在目标代码中生成一个函数，避免了通用引用的种种问题。但是毕竟开销会比其他两种方式更高，而且下面还会讨论，还会存在一些目前我们并未讨论到的开销。

2. 仅考虑对于可拷贝形参使用按值传递。不符合此条件的的形参必须有只可移动的类型（move-only types）（的数据成员），因为按值传递的函数总是会生成形参作为实参副本，而如果它们不可拷贝，副本就必须通过移动构造函数创建。回忆一下传值方案比“重载”方案的优势在于，仅有一个函数要写。但是对于只可移动类型，没必要为左值实参提供重载，因为拷贝左值需要拷贝构造函数，只可移动类型的拷贝构造函数是禁用的。那意味着只需要支持右值实参，“重载”方案只需要一个重载函数：接受右值引用的函数（即直接写接受右值引用的函数即可）。

考虑一个类：有个 std::unique_ptr<std::string>的数据成员，对它有个赋值器（setter）。std::unique_ptr 是只可移动的类型，所以赋值器的“重载”方式只有一个函数：

```cpp
class Widget {
public:
    …
    void setPtr(std::unique_ptr<std::string>&& ptr)
    { p = std::move(ptr); }

private:
    std::unique_ptr<std::string> p;
};

Widget w;
…
w.setPtr(std::make_unique<std::string>("Modern C++"));
// 从std::make_unique返回的右值std::unique_ptr<std::string>通过右值引用被传给setPtr，然后移动到数据成员p中。整体开销就是一次移动。

// 如果按值传递
class Widget {
    public:
    ...
    void setPtr(std::unique_ptr<std::string> ptr)
    { p = std::move(ptr); }
}
// 同样的调用就会先移动构造ptr形参，然后ptr再移动赋值到数据成员p，整体开销就是两次移动——是“重载”方法开销的两倍。
```

3. 按值传递应该仅考虑那些移动开销小的形参。当移动的开销较低，额外的一次移动才能被开发者接受，但是当移动的开销很大，执行不必要的移动就类似执行一个不必要的拷贝，而避免不必要的拷贝的重要性就是最开始 C++98 规则中避免传值的原因！

4. 你应该只对总是被拷贝的形参考虑按值传递。

为了看清楚为什么这很重要，假定在拷贝形参到 names 容器前，addName 需要检查新的名字的长度是否过长或者过短，如果是，就忽略增加名字的操作：

```cpp
class Widget {
public:
    void addName(std::string newName)
    {
        if ((newName.length() >= minLen) &&
            (newName.length() <= maxLen))
        {
            names.push_back(std::move(newName));
        }
    }
    …
private:
 std::vector<std::string> names;
};
```

即使这个函数没有在 names 添加任何内容，也增加了构造和销毁 newName 的开销，而按引用传递会避免这笔开销。

即使你编写的函数对可拷贝类型执行无条件的复制，且这个类型移动开销小，有时也可能不适合按值传递。这是因为函数拷贝一个形参存在两种方式：一种是通过构造函数（拷贝构造或者移动构造），还有一种是赋值（拷贝赋值或者移动赋值）。addName 使用构造函数，它的形参 newName 传递给 vector::push_back，在这个函数内部，newName 是通过拷贝构造在 std::vector 末尾创建一个新元素。对于使用构造函数拷贝形参的函数，之前的分析已经可以给出最终结论：按值传递对于左值和右值均增加了一次移动操作的开销。

当形参通过赋值操作进行拷贝时，分析起来更加复杂。比如，我们有一个表征密码的类，因为密码可能会被修改，我们提供了赋值器函数 changeTo。用按值传递的策略，我们实现一个 Password 类如下：

```cpp
class Password {
public:
    explicit Password(std::string pwd)      //传值
    : text(std::move(pwd)) {}               //构造text
    void changeTo(std::string newPwd)       //传值
    { text = std::move(newPwd); }           //赋值text
    …
private:
    std::string text;                       //密码的text
};
std::string initPwd("Supercalifragilisticexpialidocious");
Password p(initPwd);
```

毫无疑问：p.text 被给定的密码构造，在构造函数用按值传递的方式增加了一次移动构造的开销，如果使用重载或者通用引用就会消除这个开销。一切都还好。

但是，该程序的用户可能对这个密码不太满意，因为“Supercalifragilisticexpialidocious”可以在许多字典中找到。他或者她因此采取等价于如下代码的一些动作：

```cpp
std::string newPassword = "Beware the Jabberwock";
p.changeTo(newPassword);
```

changeTo 使用赋值来拷贝形参 newPwd，可能导致函数的按值传递实现方案的开销大大增加。

传递给 changeTo 的实参是一个左值（newPassword），所以 newPwd 形参被构造时，std::string 的拷贝构造函数会被调用，这个函数会分配新的存储空间给新密码。newPwd 会移动赋值到 text，这会导致 text 本来持有的内存被释放。所以 changeTo 存在两次动态内存管理的操作：一次是为新密码创建内存，一次是销毁旧密码的内存。

但是在这个例子中，旧密码比新密码长度更长，所以不需要分配新内存，销毁旧内存的操作。如果使用引用的方式，有可能两次动态内存管理操作得以避免：

```cpp
class Password {
public:
    …
    void changeTo(std::string& newPwd)      //对左值的重载
    {
        text = newPwd;                      //如果text.capacity() >= newPwd.size()，
                                            //可能重用text的内存
    }
    …
private:
    std::string text;                       //同上
};
```

这种情况下，按值传递的开销包括了内存分配和内存销毁——可能会比 std::string 的移动操作高出几个数量级。

如果旧密码短于新密码，在赋值过程中就不可避免要销毁、分配内存，这种情况，按值传递跟按引用传递的效率是一样的。因此，基于赋值的形参拷贝操作开销取决于具体的实参的值，这种分析适用于**在动态分配内存中存值的形参类型**。不是所有类型都满足，但是很多——包括 std::string 和 std::vector——是这样。

这种潜在的开销增加仅在传递左值实参时才适用，因为执行内存分配和释放通常发生在真正的拷贝操作（即，不是移动）中。对右值实参，移动几乎就足够了。

结论是，使用通过赋值拷贝一个形参进行按值传递的函数的额外开销，取决于传递的类型，左值和右值的比例，这个类型是否需要动态分配内存，以及，如果需要分配内存的话，赋值操作符的具体实现，还有赋值目标占的内存至少要跟赋值源占的内存一样大。对于 std::string 来说，开销还取决于实现是否使用了小字符串优化（SSO——参考[Item29](./rvalue_references%26more.md)），如果是，那么要赋值的值是否匹配 SSO 缓冲区。

所以，正如我所说，当形参通过赋值进行拷贝时，分析按值传递的开销是复杂的。通常，最有效的经验就是“在证明没问题之前假设有问题”，就是除非已证明按值传递会为你需要的形参类型产生可接受的执行效率，否则使用重载或者通用引用的实现方式。

到此为止，对于需要运行尽可能快的软件来说，按值传递可能不是一个好策略，因为避免即使开销很小的移动操作也非常重要。此外，有时并不能清楚知道会发生多少次移动操作。在 Widget::addName 例子中，按值传递仅多了一次移动操作，但是如果 Widget::addName 调用了 Widget::validateName，这个函数也是按值传递。（假定它有理由总是拷贝它的形参，比如把验证过的所有值存在一个数据结构中。）并假设 validateName 调用了第三个函数，也是按值传递……

可以看到这将会通向何方。在调用链中，每个函数都使用传值，因为“只多了一次移动的开销”，但是整个调用链总体就会产生无法忍受的开销，通过引用传递，调用链不会增加这种开销。

跟性能无关，但总是需要考虑的是，按值传递会收到切片问题的影响（不像按引用传递）。这是 C++98 的事，在此不在详述，但是如果要设计一个函数，来处理这样的形参：基类或者任何其派生类，你肯定不想声明一个那个类型的传值形参，因为你会“切掉”传入的任意派生类对象的派生类特征：

```cpp
class Widget { … };                         //基类
class SpecialWidget: public Widget { … };   //派生类
void processWidget(Widget w);               //对任意类型的Widget的函数，包括派生类型
…                                           //遇到对象切片问题
SpecialWidget sw;
…
processWidget(sw);                          //processWidget看到的是Widget，
                                            //不是SpecialWidget！
```

如果不熟悉对象切片问题，可以先通过搜索引擎了解一下。这样你就知道切片问题是 C++98 中默认按值传递名声不好的另一个原因（要在效率问题的原因之上）。有充分的理由来说明为什么你学习 C++编程的第一件事就是避免用户自定义类型进行按值传递。

C++11 没有从根本上改变 C++98 对于按值传递的智慧。通常，按值传递仍然会带来你希望避免的性能下降，而且按值传递会导致切片问题。C++11 中新的功能是区分了左值和右值实参。利用对于可拷贝类型的右值的移动语义，需要重载或者通用引用，尽管两者都有其缺陷。对于特殊的场景，可拷贝且移动开销小的类型，传递给总是会拷贝他们的一个函数，并且切片也不需要考虑，这时，按值传递就提供了一种简单的实现方式，效率接近传递引用的函数，但是避免了传引用方案的缺点。

## Item 41:remember

- 对于可拷贝，移动开销低，而且无条件被拷贝的形参，按值传递效率基本与按引用传递效率一致，而且易于实现，还生成更少的目标代码。
- 通过构造拷贝形参可能比通过赋值拷贝形参开销大的多。
- 按值传递会引起切片问题，所说不适合基类形参类型。

## Item 42:考虑就地创建而非插入 由 @wendajiang 贡献

如果你拥有一个容器，例如放着 std::string，那么当你通过插入（insertion）函数（例如 insert，push_front，push_back，或者对于 std::forward_list 来说是 insert_after）添加新元素时，你传入的元素类型应该是 std::string。毕竟，这就是容器里的内容。

逻辑上看来如此，但是并非总是如此。考虑如下代码：

```cpp
std::vector<std::string> vs;        //std::string的容器
vs.push_back("xyzzy");              //添加字符串字面量
```

这里，容器里内容是 std::string，但是实际上试图通过 push_back 加入的是字符串字面量，即引号内的字符序列。字符串字面量并不是 std::string，这意味着你传递给 push_back 的实参并不是容器里的内容类型。

std::vector 的 push_back 被按左值和右值分别重载：

```cpp
template <class T,                  //来自C++11标准
          class Allocator = allocator<T>>
class vector {
public:
    …
    void push_back(const T& x);     //插入左值
    void push_back(T&& x);          //插入右值
    …
};
```

编译器看到实参类型（const char[6]）和 push_back 采用的形参类型（std::string 的引用）之间不匹配。它们通过从字符串字面量创建一个 std::string 类型的临时对象来消除不匹配，然后传递临时变量给 push_back。换句话说，编译器处理的这个调用应该像这样

```cpp
vs.push_back(std::string("xyzzy"));
```

代码可以编译并运行，但是这份代码不如预期的执行效率高。

为了在 std::string 容器中创建新元素，调用了 std::string 的构造函数，但是这份代码并不仅调用了一次构造函数，而是调用了两次，而且还调用了 std::string 析构函数。下面是在 push_back 运行时发生了什么：

1. 一个 std::string 的临时对象从字面量“xyzzy”被创建。这个对象没有名字，我们可以称为 temp。temp 的构造是第一次 std::string 构造。因为是临时变量，所以 temp 是右值。
2. temp 被传递给 push_back 的右值重载函数，绑定到右值引用形参 x。在 std::vector 的内存中一个 x 的副本被创建。这次构造——也是第二次构造（移动构造，通过 x 移动构造该副本）——在 std::vector 内部真正创建一个对象。（将 x 副本拷贝到 std::vector 内部的构造函数是移动构造函数，因为 x 在它被拷贝前被转换为一个右值，成为右值引用。有关将右值引用形参强制转换为右值的信息，请参见 [Item25](./rvalue_references%26more.md)）。
3. 在 push_back 返回之后，temp 立刻被销毁，调用了一次 std::string 的析构函数。

这样就会有多余的临时变量的创建和销毁的开销，如果使用 emplace_back 就不会有这种问题，emplace_back 使用传递给它的任何实参直接在 std::vector 内部构造一个 std::string。没有临时变量会生成：

```cpp
vs.emplace_back("xyzzy");           //直接用“xyzzy”在vs内构造std::string
```

emplace_back 使用完美转发，因此只要你没有遇到完美转发的限制（参见 [Item30](./rvalue_references%26more.md)），就可以传递任何实参以及组合到 emplace_back。比如，如果你想通过接受一个字符和一个数量的 std::string 构造函数，在 vs 中创建一个 std::string，代码如下：

```cpp
vs.emplace_back(50, 'x');           //插入由50个“x”组成的一个std::string
```

emplace_back 可以用于每个支持 push_back 的标准容器。类似的，每个支持 push_front 的标准容器都支持 emplace_front。每个支持 insert（除了 std::forward_list 和 std::array）的标准容器支持 emplace。关联容器提供 emplace_hint 来补充接受“hint”迭代器的 insert 函数，std::forward_list 有 emplace_after 来匹配 insert_after。

使得置入（emplacement）函数功能优于插入函数的原因是它们有灵活的接口。插入函数接受对象去插入，而置入函数接受对象的构造函数接受的实参去插入。这种差异允许置入函数避免插入函数所必需的临时对象的创建和销毁。

因为可以传递容器内元素类型的实参给置入函数（因此该实参使函数执行复制或者移动构造函数），所以在插入函数不会构造临时对象的情况，也可以使用置入函数。在这种情况下，插入和置入函数做的是同一件事，比如：

```cpp
std::string queenOfDisco("Donna Summer");
vs.push_back(queenOfDisco);         //拷贝构造queenOfDisco
vs.emplace_back(queenOfDisco);      //同上
```

虽然，置入函数可以完成插入函数的所有功能。并且有时效率更高，至少在理论上，不会更低效。但是，就像说的那样，只是“理论上”。在当前标准库的实现下，有些场景，就像预期的那样，置入执行性能优于插入，但是，有些场景反而插入更快。这种场景不容易描述，因为依赖于传递的实参的类型、使用的容器、置入或插入到容器中的位置、容器中类型的构造函数的异常安全性，和对于禁止重复值的容器（即 std::set，std::map，std::unordered_set，set::unordered_map）要添加的值是否已经在容器中。因此，大致的调用建议是：通过 benchmark 测试来确定置入和插入哪种更快。

当然这个结论不是很令人满意，所以总结一下，如果下列条件都能满足，置入会优于插入：

- **值是通过构造函数添加到容器，而不是直接赋值**。 例子就像本条款刚开始的那样（用“xyzzy”添加 std::string 到 std::vector 容器 vs 中），值添加到 vs 末尾——一个先前没有对象存在的地方，新值必须通过构造函数添加到 std::vector。如果回看这个例子，新值放到已经存在了对象的一个地方，那情况就完全不一样了。考虑下：

  ```cpp
  std::vector<std::string> vs;        //跟之前一样
  …                                   //添加元素到vs
  vs.emplace(vs.begin(), "xyzzy");    //添加“xyzzy”到vs头部
  ```

  对于这份代码，没有实现会在已经存在对象的位置 vs[0]构造这个添加的 std::string。而是，通过移动赋值的方式添加到需要的位置。但是移动赋值需要一个源对象，所以这意味着一个临时对象要被创建，而置入优于插入的原因就是没有临时对象的创建和销毁，所以当通过赋值操作添加元素时，置入的优势消失殆尽。

  而且，向容器添加元素是通过构造还是赋值通常取决于实现者。但是，启发式仍然是有帮助的。基于节点的容器实际上总是使用构造添加新元素，大多数标准库容器都是基于节点的。例外的容器只有 std::vector，std::deque，std::string。（std::array 也不是基于节点的，但是它不支持置入和插入，所以它与这儿无关。）在不是基于节点的容器中，你可以依靠 emplace_back 来使用构造向容器添加元素，对于 std::deque，emplace_front 也是一样的。

- **传递的实参类型与容器的初始化类型不同**。 再次强调，置入优于插入通常基于以下事实：当传递的实参不是容器保存的类型时，接口不需要创建和销毁临时对象。当将类型为 T 的对象添加到 container<T>时，没有理由期望置入比插入运行的更快，因为**不需要创建临时对象来满足插入的接口**。

- **容器不拒绝重复项作为新值**。 这意味着容器要么允许添加重复值，要么你添加的元素大部分都是不重复的。这样要求的原因是为了判断一个元素是否已经存在于容器中，置入实现通常会创建一个具有新值的节点，以便可以将该节点的值与现有容器中节点的值进行比较。如果要添加的值不在容器中，则链接该节点。然后，如果值已经存在，置入操作取消，创建的节点被销毁，意味着构造和析构时的开销被浪费了。这样的节点更多的是为置入函数而创建，相比起为插入函数来说。

在决定是否使用置入函数时，需要注意另外两个问题。首先是资源管理。假定你有一个盛放 std::shared_ptr<Widget>s 的容器，然后你想添加一个通过自定义删除器释放的 std::shared_ptr（见[Item19](./smart_ptr.md)）

```cpp
std::list<std::shared_ptr<Widget>> ptrs;
void killWidget(Widget* pWidget);
// 插入函数代码
ptrs.push_back(std::shared_ptr<Widget>(new Widget, killWidget));
// or
ptrs.push_back({new Widget, killWidget});
```

不管哪种写法，在调用 push_back 前会生成一个临时 std::shared_ptr 对象。push_back 的形参是 std::shared_ptr 的引用，因此必须有一个 std::shared_ptr。

用 emplace_back 应该可以避免 std::shared_ptr 临时对象的创建，但是在这个场景下，临时对象值得被创建。考虑如下可能的时间序列：

1. 在上述的调用中，一个 std::shared_ptr<Widget>的临时对象被创建来持有“new Widget”返回的原始指针。称这个对象为 temp。
2. push_back 通过引用接受 temp。在存储 temp 的副本的 list 节点的内存分配过程中，内存溢出异常被抛出。
3. 随着异常从 push_back 的传播，temp 被销毁。作为唯一管理这个 Widget 的 std::shared_ptr，它自动销毁 Widget，在这里就是调用 killWidget。

这样的话，即使发生了异常，没有资源泄漏：在调用 push_back 中通过“new Widget”创建的 Widget 在 std::shared_ptr 管理下自动销毁。生命周期良好。

考虑使用 emplace_back 代替 push_back：

- 通过 new Widget 创建的原始指针完美转发给 emplace_back 中，list 节点被分配的位置。如果分配失败，还是抛出内存溢出异常。
- 当异常从 emplace_back 传播，原始指针是仅有的访问堆上 Widget 的途径，但是因为异常而丢失了，那个 Widget 的资源（以及任何它所拥有的资源）发生了泄漏。

使用带自定义删除器的 std::unique_ptr 也会有同样的问题。根本上讲，像 std::shared_ptr 和 std::unique_ptr 这样的资源管理类的高效性是以资源（比如从 new 来的原始指针）被立即传递给资源管理对象的构造函数为条件的。实际上，std::make_shared 和 std::make_unique 这样的函数自动做了这些事，是使它们如此重要的原因。

在对存储资源管理类对象的容器（比如 std::list<std::shared_ptr<Widget>>）调用插入函数时，函数的形参类型通常确保在资源的获取（比如使用 new）和资源管理对象的创建之间没有其他操作。在置入函数中，**完美转发推迟了资源管理对象的创建**（需要在函数中调用参数构建），直到可以在容器的内存中构造它们为止，这给“异常导致资源泄漏”提供了可能。所有的标准库容器都容易受到这个问题的影响。在使用资源管理对象的容器时，必须注意确保在使用置入函数而不是插入函数时，不会为提高效率带来的降低异常安全性付出代价。

坦白说，无论如何，你不应该将“new Widget”之类的表达式传递给 emplace_back 或者 push_back 或者大多数这种函数，因为，就像[Item21](./smart_ptr.md)中解释的那样，这可能导致我们刚刚讨论的异常安全性问题。消除资源泄漏可能性的方法是，使用独立语句把从“new Widget”获取的指针传递给资源管理类对象，然后这个对象作为右值传递给你本来想传递“new Widget”的函数（Item21 有这个观点的详细讨论）。使用 push_back 的代码应该如下：

```cpp
std::shared_ptr<Widget> spw(new Widget,      //创建Widget，让spw管理它
                            killWidget);
ptrs.push_back(std::move(spw));              //添加spw右值

// emplace_back
std::shared_ptr<Widget> spw(new Widget, killWidget);
ptrs.emplace_back(std::move(spw));
```

无论哪种方式，都会产生 spw 的创建和销毁成本

置入函数的第二个值得注意的方面是它们与 explicit 的构造函数的交互。鉴于 C++11 对正则表达式的支持，假设你创建了一个正则表达式对象的容器：

```cpp
std::vector<std::regex> regexes;
regexes.emplace_back(nullptr);           //添加nullptr到正则表达式的容器中？
// 如果这样写
std::regexes r = nullptr;     //错误！不能编译
regexes.emplace_back(r);      //错误！不能编译
```

当前你遇到的奇怪行为来源于“可能用字符串构造 std::regex 对象”，这就意味着下面代码合法：

```cpp
std::regex upperCaseWorld("[A-Z]+");
```

通过字符串创建 std::regex 要求相对较长的运行时开销，所以为了最小程度减少无意中产生此类开销的可能性，采用 const char\*指针的 std::regex 构造函数是 explicit 的（声明为 explicit 的构造函数不能在隐式转换中使用，即不会发生隐式转换。）。

在上面的代码中，我们要求从指针到 std::regex 的隐式转换，但是构造函数的 explicitness 拒绝了此类转换。

但是在 emplace_back 的调用中，我们没有说明要传递一个 std::regex 对象。然而，我们传递了一个 std::regex 构造函数实参。那不被认为是个隐式转换要求。相反，编译器看你像是写了如下代码：

```cpp
std::regex r(nullptr);                   //可编译
```

这个代码可以编译，但是行为不确定。使用 const char\*指针的 std::regex 构造函数要求字符串是一个有效的正则表达式，空指针并不满足要求。

注意到相似的初始化语句导致了不一样的结果：

```cpp
std::regex r1 = nullptr;                 //错误！不能编译
std::regex r2(nullptr);                  //可以编译
```

在标准的官方术语中，用于初始化 r1 的语法（使用等号）是所谓的拷贝初始化。相反，用于初始化 r2 的语法是（使用小括号，有时也用花括号）被称为直接初始化。

```cpp
using regex   = basic_regex<char>;

explicit basic_regex(const char* ptr,flag_type flags); //定义 (1)explicit构造函数

basic_regex(const basic_regex& right); //定义 (2)拷贝构造函数
```

拷贝初始化不被允许使用 explicit 构造函数：对于 r1，使用赋值运算符定义变量时将调用拷贝构造函数定义 (2)，其形参类型为 basic_regex&。因此 nullptr 首先需要隐式装换为 basic_regex（即需要调用构造函数构造实例）。而根据定义 (1)中的 explicit，这样的隐式转换不被允许，从而产生编译时期的报错。对于直接初始化，编译器会自动选择与提供的参数最匹配的构造函数，即定义 (1)（直接调用，非隐式转换）。就是初始化 r1 不能编译，而初始化 r2 可以编译的原因。

然后回到 push_back 和 emplace_back，更一般来说是，插入函数和置入函数的对比。置入函数使用直接初始化，这意味着可能使用 explicit 的构造函数。插入函数使用拷贝初始化，所以不能用 explicit 的构造函数。因此：

```cpp
regexes.emplace_back(nullptr);           //可编译。直接初始化允许使用接受指针的
                                         //std::regex的explicit构造函数
regexes.push_back(nullptr);              //错误！拷贝初始化不允许用那个构造函数
```

当你使用置入函数时，请特别小心确保传递了正确的实参，因为即使是 explicit 的构造函数也会被编译器考虑，编译器会试图以有效方式解释你的代码。

## Item 42:remember

- 原则上，置入函数有时会比插入函数高效，并且不会更差。
- 实际上，当以下条件满足时，置入函数更快：（1）值被构造到容器中，而不是直接赋值；（2）传入的类型与容器的元素类型不一致；（3）容器不拒绝已经存在的重复值。
- 置入函数可能执行插入函数拒绝的类型转换。
