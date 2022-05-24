# Smart Ptr

## Item18:对于独占资源使用 std::unique_ptr

std::unique_ptr 体现了专有所有权语义。一个 non-null std::unique_ptr 始终有其指向的内容。移动操作将所有权从源指针转移到目的指针，拷贝操作是不允许的，因为如果你能拷贝一个 std::unique_ptr ，你会得到指向相同内容的两个 std::unique_ptr ，每个都认为自己拥有资源，销毁时就会出现重复销毁。因此，std::unique_ptr 只支持移动操作。当 std::unique_ptr 销毁时，其指向的资源也执行析构函数。而原始指针需要显示调用 delete 来销毁指针指向的资源。
std::unique_ptr 常用在继承层次结构中工厂函数的返回类型：

```cpp
class Investment { ... };
class Sock: public Investment {...};
class Bond: public Investment {...};
class RealEstate: public Investment {...};
// 这种继承关系的工厂函数在堆上分配一个对象然后返回指针，调用方在不需要的时候，销毁对象
template<typename... Ts>
std::unique_ptr<Investment>
makeInvestment(Ts&&... params);
{
    ...
    auto pInvestment = makeInvestment(arguments);
    ...
} //destroy *pInvestment 出作用域销毁

```

可以在所有权转移的场景中使用它，比如将工厂返回的 std::unique_ptr 移入容器中，然后将容器元素移入对象的数据成员中，然后对象随即被销毁。发生这种情况时，并且销毁该对象将导致销毁从工厂返回的资源，对象 std::unique_ptr 的数据成员也被销毁。
还可以自定义 std::unique_ptr 在销毁时调用的函数：

```cpp
auto delInvmt = [](Investment* pInvestment) // 销毁前打印日志
{
    makeLogEntry(pInvestment);
    delete pInvestment;
};
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>
makeInvestment(Ts&& params)
{
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);
    if (/*a Stock object should be created*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* a Bond object should be created */ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* a RealEstate object should be created */ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```

使用了 unique_ptr 意味着你不需要考虑在资源释放时的路径，以及确保只释放一次，std::unique_ptr 解决了这一问题。

- delInvmt 是自定义的从 makeInvestment 返回的析构函数。所有的自定义的析构行为接受要销毁对象的原始指针，然后执行销毁操作。如上例子。使用 lambda 创建 delInvmt 是方便的，而且，正如稍后看到的，比编写常规的函数更有效
- 当使用自定义删除器时，必须将其作为第⼆个参数传给 std::unique_ptr 。对于 decltype。（decltype 查看[item3](deducing_types.md)）
- makeInvestment 的基本策略是创建一个空的 std::unique_ptr ，然后指向一个合适类型的对象，然后返回。
- 尝试将原始指针（比如 new 创建）赋值给 std::unique_ptr 通不过编译，因为不存在从原始指针到智能指针的隐式转换。这种隐式转换会出问题，所以禁止。这就是为什么通过 reset 来传递 new 指针的原因
- 使用 new 时，要使用 std::forward 作为参数来完美转发给 makeInvestment （std::forward 查看[Item25](rvalue_references%26more.md)）。这使调用者提供的所有信息可用于正在创建的对象的构造函数
- 自定义删除器的参数类型是 Investment* ，尽管真实的对象类型是在 makeInvestment 内部创建的，它最终通过在 lambda 表达式中，作为 Investment* 对象被删除。这意味着我们通过基类指针删除派生类实例，为此，基类必须是虚函数析构：

```cpp
class Investment {
public:
    ...
    virtual ~Investment();
    ...
};
// c++14
template<typename... Ts>
auto makeInvestment(Ts&& params)
{
    auto delInvmt = [](Investment* pInvestment)
    {
        makeLogEntry(pInvestment);
        delete pInvestment;
    };
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);
    if (/*a Stock object should be created*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* a Bond object should be created */ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* a RealEstate object should be created */ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }
    return pInv;
}
```

当使用默认删除器时，你可以合理假设 std::unique_ptr 和原始指针大小相同。当自定义删除器时，情况可能不再如此。删除器是个函数指针，通常会使 std::unique_ptr 的字节从一个增加到两个。对于删除器来说，大小取决于函数对象中存储的状态多少，无状态函数对象（比如没有捕获的 lambda 表达式）对大小没有影响，这意味当自定义删除器可以被 lambda 实现时，尽量使用 lambda

```cpp
auto delInvmt = [](Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
};
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>
makeInvestment(Ts&& params); //返回Investment*的大小

void delInvmt2(Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
}
template<typename... Ts>
std::unique_ptr<Investment, void(*)(Investment*)>
makeInvestment(Ts&&... params); //返回Investment*的指针加⾄少一个函数指针的大小
```

工厂函数不是 std::unique_ptr 的唯一常见用法。作为实现 Pimpl Idiom 的一种机制，它更为流行，见[Item22](#item22当使用-pimpl-惯用法请在实现⽂件中定义特殊成员函数)

数组的 std::unique_ptr 的存在应该不被使用，因为 std::array ,std::vector , std::string 这些更好用的数据容器应该取代原始数组。

std::unique_ptr 可以方便地转化为 std::shared_ptr, 通过返回 std::unique_ptr ，工厂为调用者提供了最有效的智能指针，但它们并不妨碍调用者用其更灵活的替换它。

```cpp
std::shared_ptr<Investment> sp = makeInvestment(arguments);
```

## Item18-remember

- std::unique_ptr 是轻量级、快速的、只能 move 的管理专有所有权语义资源的智能指针
- 默认情况，资源销毁通过 delete，但是支持⾃定义 delete 函数。有状态的删除器和函数指针会增加 std::unique_ptr 的大小
- 将 std::unique_ptr 转化为 std::shared_ptr 是简单的

## Item19:对于共享资源使用 std::shared_ptr

std::shared_ptr 可以让多个 std::shared_ptr 指向一个对象，std::shared_ptr 通过引用计数来确保它是否是最后一个指向某种资源的指针，引用计数关联资源并跟踪有多少 std::shared_ptr 指向该资源。 std::shared_ptr 构造函数递增引用计数值，析构函数递减值，拷⻉赋值运算符可能递增也可能递减值。（如果 sp1 和 sp2 是 std::shared_ptr 并且指向不同对象，赋值运算符 sp1=sp2 会使 sp1 指向 sp2 指向的对象。直接效果就是 sp1 引用计数减一，sp2 引用计数加一。）如果 std::shared_ptr 发现引用计数值为零，没有其他 std::shared_ptr 指向该资源，它就会销毁资源。

shared_ptr 的性能代价：

- shared_ptr 维护了两个指针，一个指向实例，一个指向**控制块**，所以他的大小是普通指针的两倍
- 引用计数需要动态分配，所以使用 shared_ptr 构造时需要两次动态分配内存，使用 make_shared 只用分配一次内存
- 为适用多线程场景，递增递减引用计数必须是原子性的
- shared_ptr 支持对同一类型的对象自定义销毁器

在进行移动构造的时候，由于之前的指针要置空，新的 std::shared_ptr 指向资源，所以引用计数不变。

shared_ptr 也支持对象自定义销毁器，但是对于 std::unique_ptr 来说，销毁器
类型是智能指针类型的一部分。对于 std::shared_ptr 则不是：

```cpp
// 自定义销毁器
auto loggingDel = [](Widget *pw)
{
    makeLogEntry(pw);
    delete pw;
};
std::unique_ptr<Widget, decltype(loggingDel)> upw(new Widget,loggingDel); //对于同一对象，不同类型的销毁器，是不同类型
std::shared_ptr<Widget> spw(new Widget, loggingDel);

// 这样的设计也使std::shared_ptr更加灵活
auto customDeleter1 = [](Widget *pw) { … };
auto customDeleter2 = [](Widget *pw) { … };
std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
std::shared_ptr<Widget> pw2(new Widget, customDeleter2);
// pw1 pw2可以放在同一类型的容器中，它们也能相互赋值，也可以传入形参为 std::shared_ptr<Widget> 的函数。
std::vector<std::shared_ptr<Widget>> vpw{ pw1, pw2 };
```

std::shared_ptr 由原始指针和控制块指针构成：

- 由于使用控制块，不同于 std::unique_ptr 的地方是，指定自定义销毁器不会改变 std::shared_ptr 对象的大小。不管销毁器是什么，一个 std::shared_ptr 对象都是两个指针大小.
- shared_ptr 内存分配如下所示，控制块中包含引用计数，弱引用，销毁器等
  ![](./shared_ptr_control_block.jpg)

控制块的创建遵循以下几条规则：

- make_shared 只能通过对象的参数构造，所以 std::make_shared 总是创建一个控制块（详见[Item21](#item21优先考虑使用-stdmakeunique-和-stdmakeshared-而非-new)）。它创建一个指向新对象的指针，所以可以肯定 std::make_shared 调用时对象不存在其他控制块。
- 当从独占指针上构造出 std::shared_ptr 时会创建控制块
- 当从原始指针上构造出 std::shared_ptr 时会创建控制块，而用 std::shared_ptr 或者 std::weak_ptr 作为构造函数实参创建 std::shared_ptr 不会创建新控制块，因为它可以依赖传递来的智能指针指向控制块。

下述代码当从原始指针上构造超过一个 std::shared_ptr 时，多个智能指针析构时会对 pw 指向的内存进行两次释放，发生错误

```cpp
auto pw = new Widget; // pw是原始指针
…
std::shared_ptr<Widget> spw1(pw, loggingDel); // 为*pw创建控制块
…
std::shared_ptr<Widget> spw2(pw, loggingDel); // 为*pw创建第⼆个控制块
```

- 由于上述原因，构建 shared_ptr 时对对象进行实例化，创建 spw2 时使用 spw1 构建即可：

```cpp
std::shared_ptr<Widget> spw1(new Widget, loggingDel);// 直接使用new的结果
std::shared_ptr<Widget> spw2(spw1); // spw2使用spw1一样的控制块

// 或者使用 make_shared 进行构建

auto spw1 = std::make_shared<Widget>();
```

- 当需要将对象的 this 当作 shared_ptr 使用时，std::shared_ptr 会由此为指向的对象（ \*this ）创建一个控制块，若果在成员函数外早已存在指向该对象的 shared_ptr，则在析构时会像上文中一样被析构两次，发生错误，所以该对象需要继承 std::enable_shared_from_this<Widget>（模板类），并使用 shared_from_this()返回一个 shared_ptr 仅增加引用计数，不创建控制块；同时为了防止客户端在调用 std::shared_ptr 前先调用 shared_from_this，需要把该对象的构造函数放在 private 中，通过工厂模式构建，这要该类仅能构造成 shared_ptr 而不是普通指针。

```cpp
// 错误的
std::vector<std::shared_ptr<Widget>> processedWidgets;
class Widget {
public:
    …
    void process();
    …
};
void Widget::process()
{
    … // 处理Widget
    processedWidgets.emplace_back(this); // 然后将他加到已处理过的Widget的列表中。这是错的！
}

class Widget: public std::enable_shared_from_this<Widget> {
public:
    // 完美转发参数的工厂方法，将构造函数设置为private
    template<typename... Ts>
    static std::shared_ptr<Widget> create(Ts&&... params); // 返回shared_ptr
    …
    void process(); // 和前⾯一样
    …
private:
    …
};
void Widget::process()
{
    // 和之前一样，处理Widget
    …
    // 把指向当前对象的shared_ptr加入processedWidgets
    processedWidgets.emplace_back(shared_from_this());
}
```

通常控制块的实现使用继承，里面还有一个虚函数（用来确保指向的对象被正确销毁）。所以对于 std::shared_ptr 的开销需要考虑：动态分配控制块，任意大小的销毁器和分配器，虚函数机制，原子引用计数修改。

在通常情况下， std::shared_ptr 创建控制块会使用默认销毁器和默认分配器，控制块只需三个 word 大小。它的分配基本上是⽆开销的。（开销被并入了指向的对象的分配成本里。细节参见[Item21](#item21优先考虑使用-stdmakeunique-和-stdmakeshared-而非-new)）。对 std::shared_ptr 解引用的开销不会比原始指针高。执行原子引用计数修改操作需要承担一两个原子操作开销，这些操作通常都会一一映射到机器指令上，所以即使对比非原子指令来说，原子指令开销较大，但是它们仍然只是单个指令。对于每个被 std::shared_ptr 指向的对象来说，控制块中的虚函数机制产生的开销通常只需要承受一次，即对象销毁的时候。

作为这些轻微开销的交换，你得到了动态分配的资源的生命周期⾃动管理的好处。大多数时候，比起手动管理，使用 std::shared_ptr 管理共享性资源都是非常合适的。

std::shared_ptr 不能处理的另一个东西是数组。和 std::unique_ptr 不同的是，std::shared_ptr 的 API 设计之初就是针对单个对象的，没有办法 std::shared_ptr<T[]> 。尽管使用自定义的删除器进行 delete[]的操作可以通过编译，但是还有一些问题：

- std::shared_ptr 没有提供 operator[] 重载，所以数组索引操作需要借助怪异的指针算术。
- std::shared_ptr 支持转换为指向基类的指针，这对于单个对象来说有效，但是当用于数组类型时相当于在类型系统上开洞。（出于这个原因， std::unique_ptr 禁止这种转换。）

## Item19-remember

- std::shared_ptr 为任意共享所有权的资源一种自动垃圾回收的便捷方式。
- 较之于 std::unique_ptr ， std::shared_ptr 对象通常大两倍，控制块会产生开销，需要原子引用计数修改操作。
- 默认资源销毁是通过 delete，但是也支持自定义销毁器。销毁器的类型是什么对于 std::shared_ptr 的类型没有影响。
- 避免从原始指针变量上创建 std::shared_ptr

## Item20:当 std::shard_ptr 可能悬空时使用 std::weak_ptr

## Item20-remember

## Item21:优先考虑使用 std::make_unique 和 std::make_shared 而非 new

## Item21-remember

## Item22:当使用 Pimpl 惯用法，请在实现⽂件中定义特殊成员函数

## Item22-remember
