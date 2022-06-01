# lambda 表达式

lambda 是创建函数对象相当便捷的一种方法，对于日常的 C++开发影响是巨大的。没有 lambda 时，STL 中的“\_if”算法（比如，std::find_if，std::remove_if，std::count_if 等）通常需要繁琐的谓词，但是当有 lambda 可用时，这些算法使用起来就变得相当方便。用比较函数（比如，std::sort，std::nth_element，std::lower_bound 等）来自定义算法也是同样方便的。在 STL 外，lambda 可以快速创建 std::unique_ptr 和 std::shared_ptr 的自定义删除器（见 Item18 和 19），并且使线程 API 中条件变量的谓词指定变得同样简单（参见 Item39）。除了标准库，lambda 有利于即时的回调函数，接口适配函数和特定上下文中的一次性函数。lambda 确实使 C++成为更令人愉快的编程语言。

- lambda 表达式（lambda expression）就是一个表达式：

```cpp
std::find_if(container.begin(), container.end(),
             [](int val){ return 0 < val && val < 10; });
```

- 闭包（enclosure）是 lambda 创建的运行时对象。依赖捕获模式，闭包持有被捕获数据的副本或者引用。在上面的 std::find_if 调用中，闭包是作为第三个实参在运行时传递给 std::find_if 的对象。

- 闭包类（closure class）是闭包实例化所属的类。每个 lambda 都会使编译器生成唯一的闭包类。lambda 中的语句成为其闭包类的成员函数中的可执行指令。

lambda 通常被用来创建闭包，该闭包仅用作函数的实参。上面对 std::find_if 的调用就是这种情况。然而，闭包通常可以拷贝，所以可能有多个闭包对应于一个 lambda。比如下面的代码：

```cpp
{
    int x;                                  //x是局部对象
    …

    auto c1 =                               //c1是lambda产生的闭包的副本
        [x](int y) { return x * y > 55; };

    auto c2 = c1;                           //c2是c1的拷贝

    auto c3 = c2;                           //c3是c2的拷贝
    …
}
```

## Item31:避免使用默认捕获模式

C++11 中有两种默认的捕获模式：按引用捕获和按值捕获。但默认按引用捕获模式可能会带来悬空引用的问题，而默认按值捕获模式可能会诱骗你让你以为能解决悬空引用的问题（实际上并没有），还会让你以为你的闭包是独立的（事实上也不是独立的）。

按引用捕获会导致闭包中包含了对某个局部变量或者形参的引用，变量或形参只在定义 lambda 的作用域中可用。如果该 lambda 创建的闭包生命周期超过了局部变量或者形参的生命周期，那么闭包中的引用将会变成悬空引用。举个例子，假如我们有元素是过滤函数（filtering function）的一个容器，该函数接受一个 int，并返回一个 bool，该 bool 的结果表示传入的值是否满足过滤条件：

```cpp
using FilterContainer =                     //“using”参见条款9，
	std::vector<std::function<bool(int)>>;  //std::function参见条款2

FilterContainer filters;                    //过滤函数
// 添加一个过滤器，用来过滤掉5的倍数
filters.emplace_back(                       //emplace_back的信息见条款42
	[](int value) { return value % 5 == 0; }
);

// 然而我们可能需要的是能够在运行期计算除数
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back(                               //危险！对divisor的引用
        [&](int value) { return value % divisor == 0; } //将会悬空！
    );
}
```

## Item31-remember

## Item32:使用初始化捕获来移动对象到闭包中

## Item32-remember

## Item33:对于 std::forward 的 auto&&形参使用 decltype

## Item33-remember

## Item34:优先考虑 lambda 表达式而非 std::bind

## Item34-remember

```

```
