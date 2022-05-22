# Smart Ptr
## Item18:对于独占资源使用std::unique_ptr

## Item19-remember

## Item19:对于共享资源使用std::shared_ptr
- shared_ptr维护了两个指针，一个指向实例，一个指向**控制块**，所以他的大小是普通指针的两倍
- 引用计数需要动态分配，所以使用shared_ptr构造时需要两次动态分配内存，使用make_shared只用分配一次内存
- 为适用多线程场景，递增递减引用计数必须是原⼦性的
- shared_ptr支持对同一类型的对象自定义销毁器
```cpp
std::unique_ptr<Widget, decltype(loggingDel)> upw(new Widget, loggingDel); //对于同一对象，不同类型的销毁器，是不同类型
std::shared_ptr<Widget> spw(new Widget, loggingDel);
```
- 由于使用控制块，不同于 std::unique_ptr 的地⽅是，指定⾃定义销毁器不会改变 std::shared_ptr 对象的⼤小。不管销毁器是什么，⼀个std::shared_ptr 对象都是两个指针⼤小.
- shared_ptr内存分配如下所示，控制块中包含引用计数，弱引用，销毁器等
![](./shared_ptr_control_block.jpg)
- make_shared只能通过对象的参数构造，所以std::make_shared 总是创建⼀个控制块。它创建⼀个指向新对象的指针，所以可以肯定 std::make_shared 调用时对象不存在其他控制块。
- 当从独占指针上构造出 std::shared_ptr 时会创建控制块
- 当从原始指针上构造出 std::shared_ptr 时会创建控制块，而用 std::shared_ptr 或者 std::weak_ptr 作为构造函数实参创建 std::shared_ptr 不会创建新控制块，因为它可以依赖传递来的智能指针指向控制块。下述代码当两个智能指针析构时会对pw指向的内存进行两次释放，发生错误
```cpp
auto pw = new Widget; // pw是原始指针
…
std::shared_ptr<Widget> spw1(pw, loggingDel); // 为*pw创建控制块
…
std::shared_ptr<Widget> spw2(pw, loggingDel); // 为*pw创建第⼆个控制块
```
- 由于上述原因，构建shared_ptr时对对象进行实例化，创建spw2时使用spw1构建即
```cpp
std::shared_ptr<Widget> spw1(new Widget, loggingDel);// 直接使用new的结果
std::shared_ptr<Widget> spw2(spw1); // spw2使用spw1⼀样的控制块
```
- 或者使用make_shared进行构建
```cpp
auto spw1 = std::make_shared<Widget>();
```
- 当需要将对象的this当作shared_ptr使用时，std::shared_ptr 会由此为指向的对象（ *this ）创建⼀个控制块，在析构时会像上文中一样被析构两次，发生错误，所以该对象需要继承std::enable_shared_from_this<Widget>（模板类），并使用shared_from_this()返回一个shared_ptr仅增加引用计数，不创建控制块；同时为了防⽌客⼾端在调用 std::shared_ptr 前先调用 shared_from_this，需要把该对象的构造函数放在private中，通过工厂模式构建，这要该类仅能构造成shared_ptr而不是普通指针。
```cpp
class Widget: public std::enable_shared_from_this<Widget> {
public:
// 完美转发参数的⼯⼚⽅法
template<typename... Ts>
static std::shared_ptr<Widget> create(Ts&&... params); // 返回shared_ptr
…
void process(); // 和前⾯⼀样
…
private:
…
};
```
## Item19-remember
- std::shared_ptr 为任意共享所有权的资源⼀种⾃动垃圾回收的便捷⽅式。
- 较之于 std::unique_ptr ， std::shared_ptr 对象通常⼤两倍，控制块会产⽣开销，需要原⼦引用计数修改操作。
- 默认资源销毁是通过delete，但是也⽀持⾃定义销毁器。销毁器的类型是什么对于std::shared_ptr 的类型没有影响。
- 避免从原始指针变量上创建 std::shared_ptr
## Item20:当std::shard_ptr可能悬空时使用std::weak_ptr

## Item20-remember