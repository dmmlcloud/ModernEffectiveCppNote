#  Deducing Types
## 条款⼀:理解模板类型推导
```cpp
template<typename T>
void f(ParamType param);
f(expr); //使⽤表达式调⽤f
```
在编译期间，编译器使⽤expr进⾏两个类型推导：⼀个是针对T的，另⼀个是针对ParamType的，例如：
```cpp
template<typename T>
void f(const T& param);
int x = 0;
f(x); //⽤⼀个int类型的变量调⽤f
```
此时T被推导为int， ParamType 被推导为const int &

T的推导不仅取决于expr的类型，也取决于ParamType的类型。这⾥有三种情况：
- ParamType是⼀个指针或引⽤，但不是通⽤引⽤（关于通⽤引⽤请参⻅Item24。在这⾥你只需要
知道它存在，而且不同于左值引⽤和右值引⽤）
- ParamType⼀个通⽤引⽤
- ParamType既不是指针也不是引⽤
### ParamType是⼀个指针或引⽤，但不是通⽤引⽤
- **如果expr的类型是⼀个引⽤，忽略引⽤部分**
- 然后剩下的部分决定T，然后T与形参匹配得出最终ParamType
```cpp
template<typename T>
void f(T & param); //param是⼀个引⽤

int x=27; //x是int
const int cx=x; //cx是const int
const int & rx=cx; //rx是指向const int的引⽤

f(x); //T是int，param的类型是int&
f(cx); //T是const int，param的类型是const int &
f(rx); //T是const int，param的类型是const int &
```
由cx，rx看出类型推断会保留其常量属性；

是类型推导会如左值引⽤⼀样对待右值引⽤；

### ParamType⼀个通⽤引⽤
- 如果expr是左值，T和ParamType都会被推导为左值引⽤。这⾮常不寻常，第⼀，这是模板类型推
导中唯⼀⼀种T和ParamType都被推导为引⽤的情况。第⼆，虽然ParamType被声明为右值引⽤类
型，但是最后推导的结果它是左值引⽤。
- 如果expr是右值，就使⽤情景⼀的推导规则
```cpp
template<typename T>
void f(T&& param); //param现在是⼀个通⽤引⽤类型

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
即如果expr为左值，则T和param都为左值引用，若expr为右值，则T为普通类型，param为右值引用类型

### ParamType既不是指针也不是引⽤
以值传递的方式处理，⽆论传递什么param都会成为它的⼀份拷⻉——⼀个完整的新对象。事实上param成为⼀个新对象这⼀⾏为会影响T如何从expr中推导出结果。
- 和之前⼀样，如果expr的类型是⼀个引⽤，忽略这个引⽤部分
- 如果忽略引⽤之后expr是⼀个const，那就再忽略const。如果它是volatile，也会被忽略（volatile
不常⻅，它通常⽤于驱动程序的开发中。关于volatile的细节请参⻅Item40)
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
指向const的指针和引用时，const都会保留（因为拷贝的是指针或者引用），而指针的常量属性（如果有）则会被忽略
```cpp
const char* const ptr = //ptr是⼀个常量指针，指向常量对象
" Fun with pointers"; // param是const char*
```
对于数组，由于c语言的基础，函数调用时，它和指针时等价的，因此对于不是指针和引用的ParamType，T会被推断为指针
```cpp
void myFunc(int param[]);
void myFunc(int *param); //同上

const char name[] = "J. P. Briggs";
template<typename T>
void f(T param);
f(name); //T会被推导为const char*
```
但是虽然函数不能接受真正的数组，但是可以接受指向数组的引⽤
```cpp
template<typename T>
void f(T& param);
f(name); //传数组
```
此时T被推导为const char[13]，param则被推导为const char(&)[13]，他可以帮助我们在模板函数内获得数组的大小
```cpp
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept // 将⼀个函数声明为constexpr使得结果在编译期间可⽤
{
    return N;
}

int keyVals[] = {1,3,5,7,9,11,22,25}; //keyVals有七个元素，由于constexpr
int mappedVals[arraySize(keyVals)]; //mappedVals也有七个
```

在C++中不⽌是数组会退化为指针，函数类型也会退化为⼀个函数指针，我们对于数组的全部讨论都可
以应⽤到函数来：
```cpp
void someFunc(int, double); //someFunc是⼀个函数，类型是void(int,double)
template<typename T>
void f1(T param); //传值
template<typename T>
void f2(T & param); //传引⽤
f1(someFunc); //param被推导为指向函数的指针，类型是void(*)(int, double)
f2(someFunc); //param被推导为指向函数的引⽤，类型为void(&)(int, bouel)
```
auto依赖于模板类型推导，正如我在开始谈论的，在⼤多数情况下它们的⾏为很直接。

## Item1-remember
**记住：**
- 在模板类型推导时，有引⽤的实参会被视为⽆引⽤，他们的引⽤会被忽略
- 对于通⽤引⽤的推导，左值实参会被特殊对待
- 对于传值类型推导，实参如果具有常量性和易变性会被忽略
- 在模板类型推导时，数组或者函数实参会退化为指针，除⾮它们被⽤于初始化引⽤

## 条款⼆:理解auto类型推导
auto类型推导除了⼀个例外，其他情况都和模板类型推导⼀样
```cpp
auto x = 27;
const auto cx = x;
const auto & rx=cx;
// 等价于
template<typename T> //理想化的模板⽤来推导x的类型
void func_for_x(T param);
func_for_x(27);

template<typename T> //理想化的模板⽤来推导cx 的类型
void func_for_cx(const T param);
func_for_cx(x);

template<typename T> //理想化的模板⽤来推导rx的类型
void func_for_rx(const T & param);
func_for_rx(x)；
```
即三种情况：（类型说明符即为template中的paramType）
- 类型说明符是⼀个指针或引⽤但不是通⽤引⽤
- 类型说明符⼀个通⽤引⽤
- 类型说明符既不是指针也不是引⽤

对于数组和函数退化成指针，auto也是相同的：
```cpp
const char name[] = //name的类型是const char[13]
"R. N. Briggs";
auto arr1 = name; //arr1的类型是const char*
auto& arr2 = name; //arr2的类型是const char(&)[13]
void someFunc(int,double);
auto func1=someFunc; //func1的类型是void(*)(int,double)
auto& func2 = someFunc; //func2的类型是void(&)(int,double)
```
对于c++11，可以对int类型变量进行以下初始化：
```cpp
int x1=27;
int x2(27);
int x3={27};
int x47{27};
```
如果使用auto，则结果会有所不同
```cpp
auto x1=27; //类型是int，值是27
auto x2(27); //同上
auto x3={27}; //类型是std::initializer_list<int>,值是{27}
auto x4{27}; //同上
auto x5={1,2,3.0}; //错误！auto类型推导不能⼯作，因为std::initializer_list不能接受不同类型的变量
```
对于std::initializer_list，template无法进行推导
```cpp
auto x={11,23,9}; //x的类型是std::initializer_list<int>
template<typename T>
void f(T param);
f({11,23,9}); //错误！不能推导出T
template<typename T>
void f(std::initializer_list<T> initList);
f({11,23,9}); //T被推导为int，initList的类型被推导为std::initializer_list<int>
```
在c++14中，在返回类型和lambda函数中使用auto，由于它的工作机制时模板的机制所以：
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
- auto类型推导通常和模板类型推导相同，但是auto类型推导假定花括号初始化代表
std::initializer_list而模板类型推导不这样做
- 在C++14中auto允许出现在函数返回值或者lambda函数形参中，但是它的⼯作机制是模板类型推
导那⼀套⽅案。