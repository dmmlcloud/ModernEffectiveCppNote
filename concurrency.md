# 并发 API

C++11 的伟大成功之一是将并发整合到语言和库中，C++对于并发的大量支持是在对编译器作者约束的层面，标准库并发组件（任务 tasks，期望 futures，线程 threads，互斥 mutexes，条件变量 condition variables，原子对象 atomic objects 等）仅仅是成为并发软件开发者丰富工具集的基础。

在接下来的条款中，记住标准库有两个 future 的模板：std::future 和 std::shared_future。在许多情况下，区别不重要，所以我们经常简单的混于一谈为 futures。

## Item 35:优先考虑基于任务的编程而非基于线程的编程

如果开发者想要异步执行 doAsyncWork 函数，通常有两种方式：

- 一是通过创建 std::thread 执行 doAsyncWork，这是应用了基于线程（thread-based）的方式：
  ```cpp
  int doAsyncWork();
  std::thread t(doAsyncWork);
  ```
- 其二是将 doAsyncWork 传递给 std::async，一种**基于任务**（task-based）的策略：
  ```cpp
  auto fut = std::async(doAsyncWork); //“fut”表示“future”
  ```
  这种方式中，传递给 std::async 的函数对象被称为一个任务（task）。

基于任务的方法通常比基于线程的方法更优。我们假设调用 doAsyncWork 的代码对于其提供的返回值是有需求的。基于线程的方法对此无能为力，而基于任务的方法就简单了，因为 std::async 返回的 future 提供了 get 函数（从而可以获取返回值）。如果 doAsycnWork 发生了异常，get 函数就显得更为重要，因为 get 函数可以提供抛出异常的访问，而基于线程的方法，如果 doAsyncWork 抛出了异常，程序会直接终止（通过调用 std::terminate）。

基于线程与基于任务最根本的区别在于，基于任务的抽象层次更高。基于任务的方式使得开发者从线程管理的细节中解放出来，对此在 C++并发软件中总结了“thread”的三种含义：

- 硬件线程（hardware threads）是真实执行计算的线程。现代计算机体系结构为每个 CPU 核心提供一个或者多个硬件线程。
- 软件线程（software threads）（也被称为系统线程（OS threads、system threads））是操作系统（假设有一个操作系统。有些嵌入式系统没有。）管理的在硬件线程上执行的线程。通常可以存在比硬件线程更多数量的软件线程，因为当软件线程被阻塞的时候（比如 I/O、同步锁或者条件变量），操作系统可以调度其他未阻塞的软件线程执行提供吞吐量。
- std::thread 是 C++执行过程的对象，并作为软件线程的句柄（handle）。有些 std::thread 对象代表“空”句柄，即没有对应软件线程，因为它们处在默认构造状态（即没有函数要执行）；有些被移动走（移动到的 std::thread 就作为这个软件线程的句柄）；有些被 join（它们要运行的函数已经运行完）；有些被 detach（它们和对应的软件线程之间的连接关系被打断）。

软件线程是有限的资源。如果开发者试图创建大于系统支持的线程数量，会抛出 std::system_error 异常。即使你编写了不抛出异常的代码，这仍然会发生，比如下面的代码，即使 doAsyncWork 是 noexcept，

```cpp
int doAsyncWork() noexcept;         //noexcept见条款14
std::thread t(doAsyncWork);         //如果没有更多线程可用，则抛出异常
```

设计良好的软件必须能有效地处理这种可能性，但是怎样做？一种方法是在当前线程执行 doAsyncWork，但是这可能会导致负载不均，而且如果当前线程是 GUI 线程，可能会导致响应时间过长的问题。另一种方法是等待某些当前运行的软件线程结束之后再创建新的 std::thread，但是仍然有可能当前运行的线程在等待 doAsyncWork 的动作（例如产生一个结果或者报告一个条件变量）。

即使没有超出软件线程的限额，仍然可能会遇到资源超额（oversubscription）的麻烦。这是一种当前准备运行的（即未阻塞的）软件线程大于硬件线程的数量的情况。情况发生时，线程调度器（操作系统的典型部分）会将软件线程时间切片，分配到硬件上。当一个软件线程的时间片执行结束，会让给另一个软件线程，此时发生上下文切换。软件线程的上下文切换会增加系统的软件线程管理开销，当软件线程安排到与上次时间片运行时不同的硬件线程上，这个开销会更高。这种情况下，（1）CPU 缓存对这个软件线程很冷淡（即几乎没有什么数据，也没有有用的操作指南）；（2）“新”软件线程的缓存数据会“污染”“旧”线程的数据，旧线程之前运行在这个核心上，而且还有可能再次在这里运行。

避免资源超额很困难，因为软件线程之于硬件线程的最佳比例取决于软件线程的执行频率，那是动态改变的，比如一个程序从 IO 密集型变成计算密集型，执行频率是会改变的。而且比例还依赖上下文切换的开销以及软件线程对于 CPU 缓存的使用效率。此外，硬件线程的数量和 CPU 缓存的细节（比如缓存多大，相应速度多少）取决于机器的体系结构，即使经过调校，在某一种机器平台避免了资源超额（而仍然保持硬件的繁忙状态），换一个其他类型的机器这个调校并不能提供较好效果的保证。

如果你把这些问题推给另一个人做，你就会变得很轻松，而使用 std::async 就做了这件事：

```cpp
auto fut = std::async(doAsyncWork); //线程管理责任交给了标准库的开发者
```

这种调用方式将线程管理的职责转交给 C++标准库的开发者。举个例子，这种调用方式会减少抛出资源超额异常的可能性，因为这个调用可能不会开启一个新的线程。你会想：“怎么可能？如果我要求比系统可以提供的更多的软件线程，创建 std::thread 和调用 std::async 为什么会有区别？”确实有区别，因为以这种形式调用（即使用默认启动策略——见[Item36](#item-36如果有异步的必要请指定-stdlaunchthreads)）时，std::async 不保证会创建新的软件线程。然而，他们允许通过调度器来将特定函数（本例中为 doAsyncWork）运行在等待此函数结果的线程上（即在对 fut 调用 get 或者 wait 的线程上），合理的调度器在系统资源超额或者线程耗尽时就会利用这个自由度。

如果考虑自己实现“在等待结果的线程上运行输出结果的函数”，之前提到了可能引出负载不均衡的问题，这问题不那么容易解决，因为应该是 std::async 和运行时的调度程序来解决这个问题而不是你。遇到负载不均衡问题时，对机器内发生的事情，运行时调度程序比你有更全面的了解，因为它管理的是所有执行过程，而不仅仅个别开发者运行的代码。

有了 std::async，GUI 线程中响应变慢仍然是个问题，因为调度器并不知道你的哪个线程有高响应要求。这种情况下，你会想通过向 std::async 传递 std::launch::async 启动策略来保证想运行函数在不同的线程上执行（见[Item36](#item-36如果有异步的必要请指定-stdlaunchthreads)）。

最前沿的线程调度器使用系统级线程池（thread pool）来避免资源超额的问题，并且通过工作窃取算法（work-stealing algorithm）来提升了跨硬件核心的负载均衡。C++标准实际上并不要求使用线程池或者工作窃取，实际上 C++11 并发规范的某些技术层面使得实现这些技术的难度可能比想象中更有挑战。不过，库开发者在标准库实现中采用了这些技术，也有理由期待这个领域会有更多进展。如果你当前的并发编程采用基于任务的方式，在这些技术发展中你会持续获得回报。相反如果你直接使用 std::thread 编程，处理线程耗尽、资源超额、负责均衡问题的责任就压在了你身上，更不用说你对这些问题的解决方法与同机器上其他程序采用的解决方案配合得好不好了。

对比基于线程的编程方式，基于任务的设计为开发者避免了手动线程管理的痛苦，并且自然提供了一种获取异步执行程序的结果（即返回值或者异常）的方式。当然，仍然存在一些场景直接使用 std::thread 会更有优势：

- 你需要访问非常基础的线程 API。C++并发 API 通常是通过操作系统提供的系统级 API（pthreads 或者 Windows threads）来实现的，系统级 API 通常会提供更加灵活的操作方式（举个例子，C++没有线程优先级和亲和性的概念）。为了提供对底层系统级线程 API 的访问，std::thread 对象提供了 native_handle 的成员函数，而 std::future（即 std::async 返回的东西）没有这种能力。
- 你需要且能够优化应用的线程使用。举个例子，你要开发一款已知执行概况的服务器软件，部署在有固定硬件特性的机器上，作为唯一的关键进程。
- 你需要实现 C++并发 API 之外的线程技术，比如，C++实现中未支持的平台的线程池。

## Item 35:remember

- std::thread API 不能直接访问异步执行的结果，如果执行函数有异常抛出，代码会终止执行。
- 基于线程的编程方式需要手动的线程耗尽、资源超额、负责均衡、平台适配性管理。
- 通过带有默认启动策略的 std::async 进行基于任务的编程方式会解决大部分问题。

## Item 36:如果有异步的必要请指定 std::launch::threads

当你调用 std::async 执行函数时（或者其他可调用对象），你通常希望异步执行函数。但是这并不一定是你要求 std::async 执行的操作。你事实上要求这个函数按照 std::async 启动策略来执行。有两种标准策略，每种都通过 std::launch 这个限域 enum 的一个枚举名表示（关于枚举的更多细节参见 [Item10](./moving2modern_cpp.md)）。假定一个函数 f 传给 std::async 来执行：

- std::launch::async 启动策略意味着 f 必须异步执行，即在不同的线程。
- std::launch::deferred 启动策略意味着 f 仅当在 std::async 返回的 future 上调用 get 或者 wait 时才执行。这表示 f 推迟到存在这样的调用时才执行。当 get 或 wait 被调用，f 会同步执行，即调用方被阻塞，直到 f 运行结束。如果 get 和 wait 都没有被调用，f 将不会被执行。（这是个简化说法。关键点不是要在其上调用 get 或 wait 的那个 future，而是 future 引用的那个共享状态。（[Item38](#item-38关注不同线程句柄析构行为) 讨论了 future 与共享状态的关系。）因为 std::future 支持移动，也可以用来构造 std::shared_future，并且因为 std::shared_future 可以被拷贝，对共享状态——对 f 传到的那个 std::async 进行调用产生的——进行引用的 future 对象，有可能与 std::async 返回的那个 future 对象不同。这非常绕口，所以经常回避这个事实，简称为在 std::async 返回的 future 上调用 get 或 wait。）

std::async 的默认启动策略——你不显式指定一个策略时它使用的那个——不是上面中任意一个，而是求或在一起的。下面的两种调用含义相同：

```cpp
auto fut1 = std::async(f);                      // 使用默认启动策略运行f
auto fut2 = std::async(std::launch::async |     // 使用async或者deferred运行f
                       std::launch::deferred,
                       f);                      // 即：默认的启动策略
```

因此默认策略允许 f 异步或者同步执行。如同 Item35 中指出，这种灵活性允许 std::async 和标准库的线程管理组件承担线程创建和销毁的责任，避免资源超额，以及平衡负载。这就是使用 std::async 并发编程如此方便的原因。

但是，使用默认启动策略的 std::async 也有一些有趣的影响。给定一个线程 t 执行此语句：

```cpp
auto fut = std::async(f);   //使用默认启动策略运行f
```

- 无法预测 f 是否会与 t 并发运行，因为 f 可能被安排延迟运行。
- 无法预测 f 是否会在与某线程相异的另一线程上执行，这个某线程在 fut 上调用 get 或 wait。如果对 fut 调用函数的线程是 t，含义就是无法预测 f 是否在异于 t 的另一线程上执行。
- 无法预测 f 是否执行，因为不能确保在程序每条路径上，都会不会在 fut 上调用 get 或者 wait。

默认启动策略的调度灵活性导致使用 thread_local 变量比较麻烦，因为这意味着如果 f 读写了线程本地存储（thread-local storage，TLS），不可能预测到哪个线程的变量被访问，f 的 TLS 可能是为单独的线程建的，也可能是为在 fut 上调用 get 或者 wait 的线程建的。

这还会影响到基于 wait 的循环使用超时机制，因为在一个延时的任务（参见[Item35](#item-35优先考虑基于任务的编程而非基于线程的编程)）上调用 wait_for 或者 wait_until 会产生 std::launch::deferred 值。意味着，以下循环看似应该最终会终止，但可能实际上永远运行：

```cpp
using namespace std::literals;      //为了使用C++14中的时间段后缀；参见条款34

void f()                            //f休眠1秒，然后返回
{
    std::this_thread::sleep_for(1s);
}

auto fut = std::async(f);           //异步运行f（理论上）

while (fut.wait_for(100ms) !=       //循环，直到f完成运行时停止...
       std::future_status::ready)   //但是有可能永远不会发生！
{
    …
}
```

如果 f 与调用 std::async 的线程并发运行（即，如果为 f 选择的启动策略是 std::launch::async），这里没有问题（假定 f 最终会执行完毕），但是如果 f 是延迟执行，fut.wait_for 将总是返回 std::future_status::deferred。这永远不等于 std::future_status::ready，循环会永远执行下去。

这种错误很容易在开发和单元测试中忽略，因为它可能在负载过高时才能显现出来。那些是使机器资源超额或者线程耗尽的条件，此时任务推迟执行才最有可能发生。毕竟，如果硬件没有资源耗尽，没有理由不安排任务并发执行。

修复也是很简单的：只需要检查与 std::async 对应的 future 是否被延迟执行即可，那样就会避免进入无限循环。不幸的是，没有直接的方法来查看 future 是否被延迟执行。相反，你必须调用一个超时函数——比如 wait_for 这种函数。在这个情况中，你不想等待任何事，只想查看返回值是否是 std::future_status::deferred，所以无须怀疑，使用 0 调用 wait_for：

```cpp
auto fut = std::async(f);               //同上

if (fut.wait_for(0s) ==                 //如果task是deferred（被延迟）状态
    std::future_status::deferred)
{
    …                                   //在fut上调用wait或get来异步调用f
} else {                                //task没有deferred（被延迟）
    while (fut.wait_for(100ms) !=       //不可能无限循环（假设f完成）
           std::future_status::ready) {
        …                               //task没deferred（被延迟），也没ready（已准备）
                                        //做并行工作直到已准备
    }
    …                                   //fut是ready（已准备）状态
}
```

这些各种考虑的结果就是，只要满足以下条件，std::async 的默认启动策略就可以使用：

- 任务不需要和执行 get 或 wait 的线程并行执行。
- 读写哪个线程的 thread_local 变量没什么问题。
- 可以保证会在 std::async 返回的 future 上调用 get 或 wait，或者该任务可能永远不会执行也可以接受。
- 使用 wait_for 或 wait_until 编码时考虑到了延迟状态。

如果上述条件任何一个都满足不了，你可能想要保证 std::async 会安排任务进行真正的异步执行。进行此操作的方法是调用时，将 std::launch::async 作为第一个实参传递：

```cpp
auto fut = std::async(std::launch::async, f);   //异步启动f的执行
```

事实上，对于一个类似 std::async 行为的函数，但是会自动使用 std::launch::async 作为启动策略的工具，拥有它会非常方便，而且编写起来很容易也使它看起来很棒。C++11 版本如下：

```cpp
template<typename F, typename... Ts>
inline
std::future<typename std::result_of<F(Ts...)>::type>
reallyAsync(F&& f, Ts&&... params)          //返回异步调用f(params...)得来的future
{
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Ts>(params)...);
}

// C++14
template<typename F, typename... Ts>
inline
decltype(auto)
reallyAsync(F&& f, Ts&&... params)
{
    return std::async(std::launch::async,
                      std::forward<F>(f),
                      std::forward<Ts>(params)...)
}
```

这个函数接受一个可调用对象 f 和 0 或多个形参 params，然后完美转发（参见 Item25）给 std::async，使用 std::launch::async 作为启动策略。就像 std::async 一样，返回 std::future 作为用 params 调用 f 得到的结果。确定结果的类型很容易，因为 type trait std::result_of 可以提供给你。reallyAsync 就像 std::async 一样使用：

```cpp
auto fut = reallyAsync(f);
```

这个版本清楚表明，reallyAsync 除了使用 std::launch::async 启动策略之外什么也没有做。

## Item 36:remember

- std::async 的默认启动策略是异步和同步执行兼有的。
- 这个灵活性导致访问 thread_locals 的不确定性，隐含了任务可能不会被执行的意思，会影响调用基于超时的 wait 的程序逻辑。
- 如果异步执行任务非常关键，则指定 std::launch::async。

## Item 37:从各个方面使得 std::threads unjoinable

每个 std::thread 对象处于两个状态之一：**可结合的**（joinable）或者**不可结合的**（unjoinable）。

joinable 的 std::thread 对应于正在运行或者可能要运行的异步执行线程。比如，对应于一个阻塞的（blocked）或者等待调度的线程的 std::thread 是可结合的，对应于运行结束的线程的 std::thread 也可以认为是可结合的。

不可结合的 std::thread 正如所期待：一个不是可结合状态的 std::thread。不可结合的 std::thread 对象包括：

- 默认构造的 std::threads。这种 std::thread 没有函数执行，因此没有对应到底层执行线程上。
- 已经被移动走的 std::thread 对象。移动的结果就是一个 std::thread 原来对应的执行线程现在对应于另一个 std::thread。
- 已经被 join 的 std::thread 。在 join 之后，std::thread 不再对应于已经运行完了的执行线程。
- 已经被 detach 的 std::thread 。detach 断开了 std::thread 对象与执行线程之间的连接。

std::thread 的可结合性如此重要的原因之一就是**当可结合的线程的析构函数被调用，程序执行会终止**。比如，假定有一个函数 doWork，使用一个过滤函数 filter，一个最大值 maxVal 作为形参。doWork 检查是否满足计算所需的条件，然后使用在 0 到 maxVal 之间的通过过滤器的所有值进行计算。如果进行过滤非常耗时，并且确定 doWork 条件是否满足也很耗时，则将两件事并发计算是很合理的。

我们希望为此采用基于任务的设计（参见 [Item35](#item-35优先考虑基于任务的编程而非基于线程的编程)），但是假设我们希望设置做过滤的线程的优先级。Item35 阐释了那需要线程的原生句柄，只能通过 std::thread 的 API 来完成；基于任务的 API（比如 future）做不到。所以最终采用基于线程而不是基于任务。

```cpp
constexpr auto tenMillion = 10000000;           //constexpr见条款15
constexpr auto tenMillion = 10'000'000;         //C++14

bool doWork(std::function<bool(int)> filter,    //返回计算是否执行；
            int maxVal = tenMillion)            //std::function见条款2
{
    std::vector<int> goodVals;                  //满足filter的值

    std::thread t([&filter, maxVal, &goodVals]  //填充goodVals
                  {
                      for (auto i = 0; i <= maxVal; ++i)
                          { if (filter(i)) goodVals.push_back(i); }
                  });

    auto nh = t.native_handle();                //使用t的原生句柄
    …                                           //来设置t的优先级

    if (conditionsAreSatisfied()) {
        t.join();                               //等t完成
        performComputation(goodVals);
        return true;                            //执行了计算
    }
    return false;                               //未执行计算
}
```

在开始运行之后设置 t 的优先级就太晚了，所以更好的设计是在挂起状态时开始 t（这样可以在执行任何计算前调整优先级[Item39](#item-39考虑对于单次事件通信使用-void)）。

返回 doWork。如果 conditionsAreSatisfied()返回 true，没什么问题，但是如果返回 false 或者抛出异常，在 doWork 结束调用 t 的析构函数时，std::thread 对象 t 会是可结合的。这造成程序执行中止。

如果不在这里进行析构，另外两种方式会更糟：

- 隐式 join 。这种情况下，std::thread 的析构函数将等待其底层的异步执行线程完成。这听起来是合理的，但是可能会导致难以追踪的异常表现。比如，如果 conditonAreStatisfied()已经返回了 false，doWork 继续等待过滤器应用于所有值就很违反直觉。

- 隐式 detach 。这种情况下，std::thread 析构函数会分离 std::thread 与其底层的线程。底层线程继续运行。听起来比 join 的方式好，但是可能导致更严重的调试问题。比如，在 doWork 中，goodVals 是通过引用捕获的局部变量。它也被 lambda 修改（通过调用 push_back）。假定，lambda 异步执行时，conditionsAreSatisfied()返回 false。这时，doWork 返回，同时局部变量（包括 goodVals）被销毁。栈被弹出，并在 doWork 的调用点继续执行线程。

  调用点之后的语句有时会进行其他函数调用，并且至少一个这样的调用可能会占用曾经被 doWork 使用的栈位置。我们调用那么一个函数 f。当 f 运行时，doWork 启动的 lambda 仍在继续异步运行。**该 lambda 可能在栈内存上调用 push_back，该内存曾属于 goodVals，但是现在是 f 的栈内存的某个位置**。这意味着对 f 来说，内存被自动修改了！想象一下调试的时候“乐趣”吧。

标准委员会认为，销毁可结合的线程如此可怕以至于实际上禁止了它

这使你有责任确保使用 std::thread 对象时，在所有的路径上超出定义所在的作用域时都是不可结合的。但是覆盖每条路径可能很复杂，可能包括自然执行通过作用域，或者通过 return，continue，break，goto 或异常跳出作用域，有太多可能的路径。

每当你想在执行跳至块之外的每条路径执行某种操作，最通用的方式就是将该操作放入局部对象的析构函数中。这些对象称为 RAII 对象（RAII objects），从 RAII 类中实例化。（RAII 全称为 “Resource Acquisition Is Initialization”（资源获得即初始化），尽管技术关键点在析构上而不是实例化上）。RAII 类在标准库中很常见。比如 STL 容器（每个容器析构函数都销毁容器中的内容物并释放内存），标准智能指针（Item18-20 解释了，std::uniqu_ptr 的析构函数调用他指向的对象的删除器，std::shared_ptr 和 std::weak_ptr 的析构函数递减引用计数），std::fstream 对象（它们的析构函数关闭对应的文件）等。但是标准库没有 std::thread 的 RAII 类，可能是因为标准委员会拒绝将 join 和 detach 作为默认选项，不知道应该怎么样完成 RAII。

幸运的是，完成自行实现的类并不难。比如，下面的类实现允许调用者指定 ThreadRAII 对象（一个 std::thread 的 RAII 对象）析构时，调用 join 或者 detach：

```cpp
class ThreadRAII {
public:
    enum class DtorAction { join, detach };     //enum class的信息见条款10

    ThreadRAII(std::thread&& t, DtorAction a)   //析构函数中对t实行a动作
    : action(a), t(std::move(t)) {}

    ~ThreadRAII()
    {                                           //可结合性测试见下
        if (t.joinable()) {
            if (action == DtorAction::join) {
                t.join();
            } else {
                t.detach();
            }
        }
    }

    std::thread& get() { return t; }            //见下

private:
    DtorAction action;
    std::thread t;
};
```

- 构造器只接受 std::thread 右值，因为我们想要把传来的 std::thread 对象移动进 ThreadRAII。（std::thread 不可以复制。）

- 构造器的形参顺序设计的符合调用者直觉（首先传递 std::thread，然后选择析构执行的动作，这比反过来更合理），但是成员初始化列表设计的匹配成员声明的顺序。将 std::thread 对象放在声明最后。在这个类中，这个顺序没什么特别之处，但是通常，可能一个数据成员的初始化依赖于另一个，因为 std::thread 对象可能会在初始化结束后就立即执行函数了，所以在最后声明是一个好习惯。这样就能保证一旦构造结束，在前面的所有数据成员都初始化完毕，可以供 std::thread 数据成员绑定的异步运行的线程安全使用。

- ThreadRAII 提供了 get 函数访问内部的 std::thread 对象。这类似于标准智能指针提供的 get 函数，可以提供访问原始指针的入口。提供 get 函数避免了 ThreadRAII 复制完整 std::thread 接口的需要，也意味着 ThreadRAII 可以在需要 std::thread 对象的上下文环境中使用。

- 在 ThreadRAII 析构函数调用 std::thread 对象 t 的成员函数之前，检查 t 是否可结合。这是必须的，因为在不可结合的 std::thread 上调用 join 或 detach 会导致未定义行为。客户端可能会构造一个 std::thread，然后用它构造一个 ThreadRAII，使用 get 获取 t，然后移动 t，或者调用 join 或 detach，每一个操作都使得 t 变为不可结合的。

在析构函数中，存在竞争，因为在 t.joinable()的执行和调用 join 或 detach 的中间，可能有其他线程改变了 t 为不可结合，你的直觉值得表扬，但是这个担心不必要。只有调用成员函数才能使 std::thread 对象从可结合变为不可结合状态，比如 join，detach 或者移动操作。在 ThreadRAII 对象析构函数调用时，不可能有其他线程在那个对象上调用成员函数（不可能同时调用析构函数）。如果同时进行调用，是在客户端代码中试图同时在一个对象上调用两个成员函数（析构函数和其他函数）（这时是存在竞争的）。然而通常，仅当所有都为 const 成员函数时，在一个对象同时调用多个成员函数才是安全的。

在 doWork 的例子上使用 ThreadRAII 的代码如下：

```cpp
bool doWork(std::function<bool(int)> filter,        //同之前一样
            int maxVal = tenMillion)
{
    std::vector<int> goodVals;                      //同之前一样

    ThreadRAII t(                                   //使用RAII对象
        std::thread([&filter, maxVal, &goodVals]
                    {
                        for (auto i = 0; i <= maxVal; ++i)
                            { if (filter(i)) goodVals.push_back(i); }
                    }),
                    ThreadRAII::DtorAction::join    //RAII动作
    );

    auto nh = t.get().native_handle();
    …

    if (conditionsAreSatisfied()) {
        t.get().join();
        performComputation(goodVals);
        return true;
    }

    return false;
}
```

这种情况下，我们选择在 ThreadRAII 的析构函数对异步执行的线程进行 join，因为在先前分析中，detach 可能导致噩梦般的调试过程。我们之前也分析了 join 可能会导致表现异常（坦率说，也可能调试困难），但是在未定义行为（detach 导致），程序终止（使用原生 std::thread 导致），或者表现异常之间选择一个后果，可能表现异常是最好的那个。

[Item39](#item-39考虑对于单次事件通信使用-void)表明了使用 ThreadRAII 来保证在 std::thread 的析构时执行 join 有时不仅可能导致程序表现异常，还可能导致程序挂起。“适当”的解决方案是此类程序应该和异步执行的 lambda 通信，告诉它不需要执行了，可以直接返回，但是 C++11 中不支持可中断线程（interruptible threads）。

Item17 说明因为 ThreadRAII 声明了一个析构函数，因此不会有编译器生成移动操作，但是没有理由 ThreadRAII 对象不能移动。如果要求编译器生成这些函数，函数的功能也正确，所以显式声明来告诉编译器自动生成也是合适的：

```cpp
class ThreadRAII {
public:
    enum class DtorAction { join, detach };         //跟之前一样

    ThreadRAII(std::thread&& t, DtorAction a)       //跟之前一样
    : action(a), t(std::move(t)) {}

    ~ThreadRAII()
    {
        …                                           //跟之前一样
    }

    ThreadRAII(ThreadRAII&&) = default;             //支持移动
    ThreadRAII& operator=(ThreadRAII&&) = default;

    std::thread& get() { return t; }                //跟之前一样

private: // as before
    DtorAction action;
    std::thread t;
};
```

## Item 37:remember

- 在所有路径上保证 thread 最终是不可结合的。
- 析构时 join 会导致难以调试的表现异常问题。
- 析构时 detach 会导致难以调试的未定义行为。
- 声明类数据成员时，最后声明 std::thread 对象。

## Item 38:关注不同线程句柄析构行为

[Item37](#item-37从各个方面使得-stdthreads-unjoinable)中说明了可结合的 std::thread 对应于执行的系统线程。未延迟（non-deferred）任务的 future（参见[Item36](#item-36如果有异步的必要请指定-stdlaunchthreads)）与系统线程有相似的关系。因此，可以将 std::thread 对象和 future 对象都视作系统线程的句柄（handles）。

std::thread 和 future 在析构时有相当不同的行为。
在[Item37](#item-37从各个方面使得-stdthreads-unjoinable)中说明，可结合的 std::thread 析构会终止你的程序，因为两个其他的替代选择——隐式 join 或者隐式 detach 都是更加糟糕的。
但是，future 的析构表现有时就像执行了隐式 join，有时又像是隐式执行了 detach，有时又没有执行这两个选择。它永远不会造成程序终止。这个线程句柄多种表现值得研究一下。

我们可以观察到实际上 future 是通信信道的一端，被调用者通过该信道将结果发送给调用者。（[Item39](#item-39考虑对于单次事件通信使用-void)说，与 future 有关的这种通信信道也可以被用于其他目的。但是对于本条款，我们只考虑它们作为这样一个机制的用法，即被调用者传送结果给调用者。）被调用者（通常是异步执行）将计算结果写入通信信道中（通常通过 std::promise 对象），调用者使用 future 读取结果。可以想象成下面的图示，虚线表示信息的流动方向：
![](./future_1.png)

被调用者会在调用者 get 相关的 future 之前执行完成，所以结果不能存储在被调用者的 std::promise。这个对象是局部的，当被调用者执行结束后，会被销毁。

结果同样不能存储在调用者的 future，因为（当然还有其他原因）std::future 可能会被用来创建 std::shared_future（这会将被调用者的结果所有权从 std::future 转移给 std::shared_future），而 std::shared_future 在 std::future 被销毁之后可能被复制很多次。鉴于不是所有的结果都可以被拷贝（即只可移动类型），并且结果的生命周期至少与最后一个引用它的 future 一样长，所以无法决定是用 future 中哪个才是被调用者用来存储结果的。

因为与被调用者关联的对象和与调用者关联的对象都不适合存储这个结果，所以必须存储在两者之外的位置。此位置称为共享状态（shared state）。共享状态通常是基于堆的对象，但是标准并未指定其类型、接口和实现。标准库的作者可以通过任何他们喜欢的方式来实现共享状态。

我们可以想象调用者，被调用者，共享状态之间关系如下图，虚线还是表示信息流方向：
![](./future_2.png)
共享状态的存在非常重要，因为 future 的析构函数取决于与 future 关联的共享状态：

- **引用了使用 std::async 启动的未延迟任务建立的那个共享状态的最后一个 future 的析构函数会阻塞住，直到任务完成。**本质上，这种 future 的析构函数对执行异步任务的线程执行了隐式的 join。
- **其他所有 future 的析构函数简单地销毁 future 对象。**对于异步执行的任务，就像对底层的线程执行 detach（相当于）。对于延迟任务来说如果这是最后一个 future，意味着这个延迟任务永远不会执行了。

上述两条规则可以概括为一个简单的“正常”行为以及一个单独的例外。正常行为是 future 析构函数销毁 future。就是这样。那意味着不 join 也不 detach，也不运行什么，只销毁 future 的数据成员（当然，还做了另一件事，就是递减了共享状态中的引用计数，这个共享状态是由引用它的 future 和被调用者的 std::promise 共同控制的。这个引用计数让库知道共享状态什么时候可以被销毁。）

而例外情况只会在下列条件都满足的时候才出现：

- 它关联到由于调用 std::async 而创建出的共享状态。
- 任务的启动策略是 std::launch::async（参见[Item36](#item-36如果有异步的必要请指定-stdlaunchthreads)），原因是运行时系统选择了该策略，或者在对 std::async 的调用中指定了该策略。
- 这个 future 是关联共享状态的最后一个 future。对于 std::future，情况总是如此，对于 std::shared_future，如果还有其他的 std::shared_future，与要被销毁的 future 引用相同的共享状态，则要被销毁的 future 遵循正常行为（即简单地销毁它的数据成员）。

只有当上面的三个条件都满足时，future 的析构函数才会在异步任务执行完之前阻塞住。

future 的 API 没办法确定是否 future 引用了一个 std::async 调用产生的共享状态，因此给定一个任意的 future 对象，无法判断会不会因等待异步任务而阻塞析构函数：

```cpp
//这个容器可能在析构函数处阻塞，因为其中可能有一个或多个future可能引用由std::async启动的
//未延迟任务创建出来的共享状态
std::vector<std::future<void>> futs;    //std::future<void>相关信息见条款39

class Widget {                          //Widget对象可能在析构函数处阻塞
public:
    …
private:
    std::shared_future<double> fut;
};
```

当然，如果你有办法知道给定的 future 不满足上面条件的任意一条（比如由于程序逻辑造成的不满足），你就可以确定析构函数不会执行“异常”行为。比如，只有通过 std::async 创建的共享状态才有资格执行“异常”行为，但是有其他创建共享状态的方式。

比如使用 std::packaged_task 进行创建，一个 std::packaged_task 对象通过包覆（wrapping）方式准备一个函数（或者其他可调用对象）来异步执行，然后将其结果放入共享状态中。然后通过 std::packaged_task 的 get_future 函数可以获取有关该共享状态的 future：

```cpp
int calcValue();                //要运行的函数
std::packaged_task<int()>       //包覆calcValue以异步运行
    pt(calcValue);
auto fut = pt.get_future();     //从pt获取future
```

此时，我们知道 future 没有关联 std::async 创建的共享状态，所以析构函数肯定正常方式执行。

std::packaged_task 类型的 pt 可以绑定在一个线程（std::thread）上执行。（也可以通过调用 std::async 运行，但是如果你想使用 std::async 运行任务，没有理由使用 std::packaged_task，因为可以直接 std::async 进行执行。）

std::packaged_task 不可拷贝，所以当 pt 被传递给 std::thread 构造函数时，必须先转为右值（通过 std::move，参见 Item23）：

```cpp
{                                   //开始代码块
    std::packaged_task<int()>
        pt(calcValue);

    auto fut = pt.get_future();

    std::thread t(std::move(pt));   //见下
    …
}                                   //结束代码块
```

该线程 t 在程序中：

- 对 t 什么也不做。这种情况，t 会在语句块结束时是可结合的，这会使得程序终止（参见 [Item37](#item-37从各个方面使得-stdthreads-unjoinable)）。
- 对 t 调用 join。这种情况，不需要 fut 在它的析构函数处阻塞，因为 join 被显式调用了。
- 对 t 调用 detach。这种情况，不需要在 fut 的析构函数执行 detach，因为显式调用了。

换句话说，当你有一个关联了 std::packaged_task 创建的共享状态的 future 时，不需要采取特殊的销毁策略（直接销毁 future 即可），因为通常你会代码中做终止、结合或分离这些决定之一，来操作 std::packaged_task 的运行所在的那个 std::thread。

## Item 38:remember

- future 的正常析构行为就是销毁 future 本身的数据成员。
- 引用了使用 std::async 启动的未延迟任务建立的 future 创建的共享状态的最后一个 future 的析构函数会阻塞住，直到任务完成。

## Item 39:考虑对于单次事件通信使用 void

## Item 39:remember

## Item 40:对于并发使用 std::atomic，volatile 用于特殊内存区

## Item 40:remember
