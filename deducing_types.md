# Deducing Types

## 条款一:理解模板类型推导

```cpp
template<typename T>
void f(ParamType param);
f(expr); //使用表达式调用f
```

在编译期间，编译器使用 expr 进⾏两个类型推导：⼀个是针对 T 的，另⼀个是针对 ParamType 的，例如：

```cpp
template<typename T>
void f(const T& param);
int x = 0;
f(x); //用⼀个int类型的变量调用f
```

此时 T 被推导为 int， ParamType 被推导为 const int &
**ParamType 推断出来的是 param 在参数列表中的类型，param 在函数内部表现一定是左值**
T 的推导不仅取决于 expr 的类型，也取决于 ParamType 的类型。这⾥有三种情况：

- ParamType 是⼀个指针或引用，但不是通用引用（关于通用引用请参⻅ Item24。在这⾥你只需要知道它存在，而且不同于左值引用和右值引用）
- ParamType ⼀个通用引用
- ParamType 既不是指针也不是引用

### ParamType 是⼀个指针或引用，但不是通用引用

- **如果 expr 的类型是⼀个引用，忽略引用部分**
- 然后剩下的部分决定 T，然后 T 与形参匹配得出最终 ParamType

```cpp
template<typename T>
void f(T & param); //param是⼀个引用

int x=27; //x是int
const int cx=x; //cx是const int
const int & rx=cx; //rx是指向const int的引用

f(x); //T是int，param的类型是int&
f(cx); //T是const int，param的类型是const int &
f(rx); //T是const int，param的类型是const int &
```

由 cx，rx 看出类型推断会保留其常量属性；

是类型推导会如左值引用⼀样对待右值引用；

### ParamType ⼀个通用引用

- 如果 expr 是左值，T 和 ParamType 都会被推导为左值引用。这⾮常不寻常，第⼀，这是模板类型推导中唯⼀⼀种 T 和 ParamType 都被推导为引用的情况。第⼆，虽然 ParamType 被声明为右值引用类型，但是最后推导的结果它是左值引用。
- 如果 expr 是右值，就使用情景⼀的推导规则

```cpp
template<typename T>
void f(T&& param); //param现在是⼀个通用引用类型

int x=27; //如之前⼀样
const int cx=x; //如之前⼀样
const int & rx=cx; //如之前⼀样
f(x); //x是左值，所以T是int&
//param类型也是int&
f(cx); //cx是左值，所以T是const int &
//param类型也是const int&
f(rx); //rx是左值，所以T是const int &
//param类型也是const int&
f(27); //27是右值，所以T是int
//param类型就是int&&
```

即如果 expr 为左值，则 T 和 param 都为左值引用，若 expr 为右值，则 T 为普通类型，param 为右值引用类型

### ParamType 既不是指针也不是引用

以值传递的方式处理，⽆论传递什么 param 都会成为它的⼀份拷⻉——⼀个完整的新对象。事实上 param 成为⼀个新对象这⼀⾏为会影响 T 如何从 expr 中推导出结果。

- 和之前⼀样，如果 expr 的类型是⼀个引用，忽略这个引用部分
- 如果忽略引用之后 expr 是⼀个 const，那就再忽略 const。如果它是 volatile，也会被忽略（volatile 不常⻅，它通常用于驱动程序的开发中。关于 volatile 的细节请参⻅ Item40)

```cpp
template<typename T>
void f(T param); //以传值的⽅式处理param

int x=27; //如之前⼀样
const int cx=x; //如之前⼀样
const int & rx=cx; //如之前⼀样
f(x); //T和param都是int
f(cx); //T和param都是int
f(rx); //T和param都是int
```

指向 const 的指针和引用时，const 都会保留（因为拷贝的是指针或者引用），而指针的常量属性（如果有）则会被忽略

```cpp
const char* const ptr = //ptr是⼀个常量指针，指向常量对象
" Fun with pointers"; // param是const char*
```

对于数组，由于 c 语言的基础，函数调用时，它和指针时等价的，因此对于不是指针和引用的 ParamType，T 会被推断为指针

```cpp
void myFunc(int param[]);
void myFunc(int *param); //同上

const char name[] = "J. P. Briggs";
template<typename T>
void f(T param);
f(name); //T会被推导为const char*
```

但是虽然函数不能接受真正的数组，但是可以接受指向数组的引用

```cpp
template<typename T>
void f(T& param);
f(name); //传数组
```

此时 T 被推导为 const char[13]，param 则被推导为 const char(&)[13]，他可以帮助我们在模板函数内获得数组的大小

```cpp
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept // 将⼀个函数声明为constexpr使得结果在编译期间可用
{
    return N;
}

int keyVals[] = {1,3,5,7,9,11,22,25}; //keyVals有七个元素，由于constexpr
int mappedVals[arraySize(keyVals)]; //mappedVals也有七个
```

在 C++中不⽌是数组会退化为指针，函数类型也会退化为⼀个函数指针，我们对于数组的全部讨论都可以应用到函数来：

```cpp
void someFunc(int, double); //someFunc是⼀个函数，类型是void(int,double)
template<typename T>
void f1(T param); //传值
template<typename T>
void f2(T & param); //传引用
f1(someFunc); //param被推导为指向函数的指针，类型是void(*)(int, double)
f2(someFunc); //param被推导为指向函数的引用，类型为void(&)(int, bouel)
```

auto 依赖于模板类型推导，正如我在开始谈论的，在⼤多数情况下它们的⾏为很直接。

## Item1-remember

**记住：**

- 在模板类型推导时，有引用的实参会被视为⽆引用，他们的引用会被忽略
- 对于通用引用的推导，左值实参会被特殊对待
- 对于传值类型推导，实参如果具有常量性和易变性会被忽略
- 在模板类型推导时，数组或者函数实参会退化为指针，除⾮它们被用于初始化引用

## 条款二:理解 auto 类型推导

auto 类型推导除了⼀个例外，其他情况都和模板类型推导⼀样

```cpp
auto x = 27;
const auto cx = x;
const auto & rx=cx;
// 等价于
template<typename T> //理想化的模板用来推导x的类型
void func_for_x(T param);
func_for_x(27);

template<typename T> //理想化的模板用来推导cx 的类型
void func_for_cx(const T param);
func_for_cx(x);

template<typename T> //理想化的模板用来推导rx的类型
void func_for_rx(const T & param);
func_for_rx(x)；
```

即三种情况：（类型说明符即为 template 中的 paramType）

- 类型说明符是⼀个指针或引用但不是通用引用
- 类型说明符⼀个通用引用
- 类型说明符既不是指针也不是引用

对于数组和函数退化成指针，auto 也是相同的：

```cpp
const char name[] = //name的类型是const char[13]
"R. N. Briggs";
auto arr1 = name; //arr1的类型是const char*
auto& arr2 = name; //arr2的类型是const char(&)[13]
void someFunc(int,double);
auto func1=someFunc; //func1的类型是void(*)(int,double)
auto& func2 = someFunc; //func2的类型是void(&)(int,double)
```

对于 c++11，可以对 int 类型变量进行以下初始化：

```cpp
int x1=27;
int x2(27);
int x3={27};
int x47{27};
```

如果使用 auto，则结果会有所不同

```cpp
auto x1=27; //类型是int，值是27
auto x2(27); //同上
auto x3={27}; //类型是std::initializer_list<int>,值是{27}
auto x4{27}; //同上
auto x5={1,2,3.0}; //错误！auto类型推导不能⼯作，因为std::initializer_list不能接受不同类型的变量
```

对于 std::initializer_list，template 无法进行推导

```cpp
auto x={11,23,9}; //x的类型是std::initializer_list<int>
template<typename T>
void f(T param);
f({11,23,9}); //错误！不能推导出T
template<typename T>
void f(std::initializer_list<T> initList);
f({11,23,9}); //T被推导为int，initList的类型被推导为std::initializer_list<int>
```

在 c++14 中，在返回类型和 lambda 函数中使用 auto，由于它的工作机制时模板的机制所以：

```cpp
auto createInitList()
{
    return {1,2,3}; //错误！推导失败
}

std::vector<int> v;
auto resetV = [&v](const auto & newValue){v=newValue;}; //C++14
...
reset({1,2,3}); //错误！推导失败

```

## Item2-remember

- auto 类型推导通常和模板类型推导相同，但是 auto 类型推导假定花括号初始化代表 std::initializer_list 而模板类型推导不这样做
- 在 C++14 中 auto 允许出现在函数返回值或者 lambda 函数形参中，但是它的⼯作机制是模板类型推导那⼀套方案。

## 条款三:理解 decltype

简单的，decltype 只是简单的返回名字或者表达式的类型：

```cpp
const int i=0; //decltype(i)是const int
bool f(const Widget& w); //decltype(w)是const Widget&
//decltype(f)是bool(const
Widget&)
struct Point{
int x; //decltype(Point::x)是int
int y; //decltype(Point::y)是int
};
template<typename T>
class Vector{
...
T& operator[](std::size_t index);
...
}
vector<int> v; //decltype(v)是vector<int>
...
if(v[0]==0) //decltype(v[0])是int&

```

在 C++11 中，decltype 最主要的用途就是用于函数模板返回类型，而这个返回类型依赖形参，举个例子，假定我们写⼀个函数，⼀个参数为容器，⼀个参数为索引值，这个函数支持使用方括号的方式访问容器中指定索引值的数据，然后在返回索引操作的结果前执行认证用户操作。函数的返回类型应该和索引操作返回的类型相同。
对⼀个 T 类型的容器使用 operator[] 通常会返回⼀个 T&对象，比如 std::deque 就是这样，但是 std::vector 有⼀个例外，对于 std::vector，operator[]不会返回 bool&，它会返回⼀个有名字的对象类型
使用 decltype 计算返回类型可以这样实现：

```cpp
/* 函数名称前⾯的auto不会做任何的类型推导⼯作。相反的，他只是暗⽰使用了C++11的尾置返回类型语法，
即在函数形参列表后⾯使用⼀个-> 符号指出函数的返回类型，尾置返回类型的好处是我们可以在函数返回类型中使用函数参数相关的信息*/
template<typename Container,typename Index>
auto authAndAccess(Container& c,Index i)
->decltype(c[i])
{
    authenticateUser();
    return c[i]; // 返回的从operate[]返回的类型
}
```

C++11 允许自动推导单⼀语句的 lambda 表达式的返回类型， C++14 扩展到允许自动推导所有的 lambda 表达式和函数，甚⾄它们内含多条语句，此时可以省略尾置返回类型，但此时 auto 会从函数实现中推导函数的返回类型

```cpp
template<typename Container,typename Index> //C++ 14版本
auto authAndAccess(Container& c,Index i)
{
    authenticateUser();
    return c[i]; // 会省去引用
}
```

而由于 operator[]返回的是引用，而 auto 根据模板的类型推断会省略掉引用，因此返回的仅是一个右值，会导致一定的问题：

```cpp
std::deque<int> d;
...
authAndAccess(d,5)=10; //认证用⼾，返回d[5]，
//然后把10赋值给它
//⽆法通过编译器！ 无法将一个右值赋值给右值
```

使用 decltype(auto)可以解决这一问题，：auto 说明符表⽰这个类型将会被推导，decltype 说明 decltype 的规则将会引用到这个推导过程中。

```cpp
template<typename Container,typename Index>
decltype(auto)
authAndAccess(Container& c,Index i)
{
    authenticateUser();
    return c[i]; // 返回引用T&
}
```

decltype(auto)也可以用在进行初始化的时候，表示通过 decltype 的规则进行类型推断

```cpp
Widget w;
const Widget& cw = w;
auto myWidget1 = cw; //auto类型推导
//myWidget1的类型为Widget
decltype(auto) myWidget2 = cw; //decltype类型推导
//myWidget2的类型是const Widget&
```

对于 authAndAccess，要让她满足右值引用，需要修改成

```cpp
template<typename Containter,typename Index>
decltype(auto) authAndAccess(Container&& c,Index i);
```

但在这个模板中不知道操纵的容器的类型是什么，而 Index 有可能是索引对象（不是简单的 int）对⼀个未知类型的对象使用传值是通常对程序的性能有极⼤的影响在这个例⼦中还会造成不必要的拷贝，还会造成对象切片行为

```cpp
template<typename Container,typename Index> //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c,Index i){
    authenticateUser();
    return std::forward<Container>(c)[i]; //
}

template<typename Container,typename Index> //最终的C++11版本
auto
authAndAccess(Container&& c,Index i)
->decltype(std::forward<Container>(c)[i])
{
authenticateUser();
return std::forward<Container>(c)[i];
}
```

decltype 对于左值表达式返回 T&

```cpp
int x = 0;
decltype(x) // x是变量名 所以为int
decltype((x)) // (x)是左值表达式，所以是int&

decltype(auto) f1()
{
    int x = 0;
    ...
    return x; //decltype(x）是int，所以f1返回int
}
decltype(auto) f2() // 注意不仅f2的返回类型不同于f1，而且它还引用了⼀个局部变量(未定义行为)
{
    int x =0;
    ...
    return (x); //decltype((x))是int&，所以f2返回int&
}
```

## Item3-remember

- decltype 总是不加修改的产生变量或者表达式的类型。
- 对于 T 类型的左值表达式，decltype 总是产出 T 的引用即 T&。
- C++14 支持 decltype(auto) ，就像 auto ⼀样，推导出类型，但是它使用自己的独特规则进行推导。
