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

## Item 37:remember

## Item 38:关注不同线程句柄析构行为

## Item 38:remember

## Item 39:考虑对于单次事件通信使用 void

## Item 39:remember

## Item 40:对于并发使用 std::atomic，volatile 用于特殊内存区

## Item 40:remember
