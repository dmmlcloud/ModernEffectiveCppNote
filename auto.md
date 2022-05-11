# auto
## 条款五:优先考虑auto而⾮显式类型声明
对一个普通变量进行声明，可能会忘记初始化
```cpp
int x = 1;
```

不使用auto对⼀个局部变量使⽤解引⽤迭代器的⽅式初始化
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
使用auto
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
对于lambda表达式也可以用auto进行表示
```cpp
auto derefUPLess = [](const std::unique_ptr<Widget> &p1, //专⽤于Widget类型的⽐较函数
const std::unique_ptr<Widget> &p2){return *p1<*p2;};
auto derefUPLess = [](const auto& p1,const auto& p2){return *p1<*p2;}; // c++14 对形参使用auto
```
如果使用std::function(std::function 是⼀个C++11标准模板库中的⼀个模板，它泛化了函数指针的概念。与函数指针只能指向函数不同， std::function 可以指向任何可调⽤对象，也就是那些像函数⼀样能进⾏调⽤的东西。当你声明函数指针时你必须指定函数类型（即函数签名），同样当你创建 std::function 对象时你也需要提供函数签名，由于它是⼀个模板所以你需要在它的模板参数⾥⾯提供)
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
了使⽤ std::function ⽐auto会消耗更多的内存。
使⽤auto除了使⽤未初始化的⽆效变量，省略冗⻓的声明类型，直接保存闭包外，它还有⼀个好处是可
以避免⼀个问题，我称之为依赖类型快捷⽅式的问题：
```cpp
std::vector<int> v;
unsigned sz = v.size();
std::vector<int>::size_type st = v.size();
```
一般认为std::vector<int>::size_type就是usigned_int，但是对于不同的机器，他们可能不同，在Windows 64-bit上 std::vector<int>::size_type 是64位， unsigned int 是32位。此时使用auto就不用考虑这样的问题：
```cpp
auto sz =v.size();
```
对于下面的代码，由于std::unordered_map的key是一个常量，下面for循环中应该写成（std::pair<const std::string,int>），否则会根据m中的元素创建临时变量，并在结束后删除这些对象
```cpp
std::unordered_map<std::string,int> m;
...
for(const std::pair<std::string,int>& p : m)
{
    ...
}

```
而使用auto可以避免这样的问题
```cpp
for(const auto & p : m)
{
    ...
}
```
这两个例⼦说明了显式的指定类型可能会导致你不像看到的类型转换。如果你使⽤auto声明⽬标变量你就不必担⼼这个问题。

使⽤auto可以帮助我们完成⼀些重构⼯作。举个例⼦，如果⼀个函数返回类型被声明为int，但是后来你认为将它声明为long会更好，调⽤它作为初始化表达式的变量会⾃动改变类型，但是如果你不使⽤auto你就不得不在源代码中挨个找到调⽤地点然后修改它们。
## Item5-remember
- auto变量必须初始化，通常它可以避免⼀些移植性和效率性的问题，也使得重构更⽅便，还能让你少打⼏个字。
- 正如Item2和6讨论的，auto类型的变量可能会踩到⼀些陷阱。

