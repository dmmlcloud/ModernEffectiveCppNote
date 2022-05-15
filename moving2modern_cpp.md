# Moving to Modern C++

## 条款七:区别使⽤()和{}创建对象
```cpp
Widget w1; //调⽤默认构造函数
Widget w2 = w1; //不是赋值运算符，调⽤拷⻉构造函数
w1 = w2; //是⼀个赋值运算符，调⽤operator=函数
```
C++11使⽤统⼀初始化,统⼀初始化是⼀个概念上的东西，而括号初始化是⼀个具体语法构型。
使⽤花括号，指定⼀个容器的元素变得很容易：
```cpp
std::vector<int> v{1,3,5}; //v包含1,3,5
```
括号初始化也能被⽤于为⾮静态数据成员指定默认初始值。C++11允许"="初始化也拥有这种能⼒：
```cpp
class Widget{
...
private:
    int x{0}; //没问题，x初始值为0
    int y = 0; //同上
    int z(0); //错误！
}
```
不可拷⻉的对象可以使⽤花括号初始化或者小括号初始化，但是不能使⽤"="初始化
```cpp
std::vector<int> ai1{0}; //没问题，x初始值为0
std::atomic<int> ai2(0); //没问题
std::atomic<int> ai3 = 0; //错误！
```
在C++中这三种⽅式都被指派为初始化表达式，但是只有括号任何地⽅都能被使⽤，所以叫统一初始化
但是括号初始化不允许内置隐式的变窄转换：
```cpp
double x,y,z;
int sum1{x+y+z}; //错误！三个double的和不能⽤来初始化int类型的变量
int sum2(x + y +z); //可以（表达式的值被截为int）
int sum3 = x + y + z; //同上
```
C++规定任何能被决议为⼀个声明的东西必须被决议为声明。这个规则的副作⽤是让很多程序员备受折磨：当他们想创建⼀个使⽤默认构造函数构造的对象，却不小⼼变成了函数声明。
```cpp
Widget w1(10); //使⽤实参10调⽤Widget的⼀个构造函数
Widget w2(); //最令⼈头疼的解析！声明⼀个函数w2，返回Widget
// 使用花括号不会
Widget w3{}; //调⽤没有参数的构造函数构造对象
```
括号初始化的缺点是有时它有⼀些令⼈惊讶的⾏为。Item2解释了当auto声明的变量使⽤花括号初始化，变量就会被推导为std::initializer_list，尽管使⽤相同内容的其他初始化⽅式会产⽣正常的结果。所以，你越喜欢⽤atuo，你就越不能⽤括号初始化。
在构造函数调⽤中，只要不包含std::initializer_list参数，那么花括号初始化和小括号初始化都会产⽣⼀
样的结果：
```cpp
class Widget {
public:
    Widget(int i, bool b); //未声明默认构造函数
    Widget(int i, double d); // std::initializer_list参数
…
};
Widget w1(10, true); // 调⽤构造函数
Widget w2{10, true}; // 同上
Widget w3(10, 5.0); // 调⽤第⼆个构造函数
Widget w4{10, 5.0}; // 同上

// 添加一个以std::initializer_list为参数的函数
// w2和w4将会使⽤新添加的构造函数构造，即使另⼀个⾮std::initializer_list构造函数对于实参是更好的选择
class Widget {
public:
    Widget(int i, bool b); // 同上
    Widget(int i, double d); // 同上
    Widget(std::initializer_list<long double> il); //新添加的
…
};
Widget w1(10, true); // 使⽤小括号初始化
//调⽤第⼀个构造函数
Widget w2{10, true}; // 使⽤花括号初始化
// 调⽤第三个构造函数
// (10 和 true 转化为long double)
Widget w3(10, 5.0); // 使⽤小括号初始化
// 调⽤第⼆个构造函数
Widget w4{10, 5.0}; // 使⽤花括号初始化
// 调⽤第三个构造函数
// (10 和 5.0 转化为long double)


// 甚⾄普通的构造函数和移动构造函数都会被std::initializer_list构造函数劫持：
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<long double> il);
    operator float() const; // convert to float
};
Widget w5(w4); // 使⽤小括号，调⽤拷⻉构造函数
Widget w6{w4}; // 使⽤花括号，调⽤std::initializer_list构造函数
Widget w7(std::move(w4)); // 使⽤小括号，调⽤移动构造函数
Widget w8{std::move(w4)}; // 使⽤花括号，调⽤std::initializer_list构造函数

Widget w{10, 5.0}; //错误！要求变窄转换

// 只有当没办法把括号初始化中实参的类型转化为std::initializer_list时，编译器才会回到正常的函数决议流程中。
class Widget {
public:
    Widget(int i, bool b);
    Widget(int i, double d);
    Widget(std::initializer_list<std::string> il);
…
};
// int, bool, double无法转化为string
Widget w1(10, true); // 使⽤小括号初始化，调⽤第⼀个构造函数
Widget w2{10, true}; // 使⽤花括号初始化，调⽤第⼀个构造函数
Widget w3(10, 5.0); // 使⽤小括号初始化，调⽤第⼆个构造函数
Widget w4{10, 5.0}; // 使⽤花括号初始化，调⽤第⼆个构造函数

class Widget {
public:
    Widget();
    Widget(std::initializer_list<int> il);
...
};
Widget w1; // 调⽤默认构造函数
Widget w2{}; // 同上
Widget w3(); // 最令⼈头疼的解析！声明⼀个函数
Widget w4({}); // 调⽤std::initializer_list
Widget w5{{}}; // 同上
```
std::vector有⼀个⾮std::initializer_list构造函数允许你去指定容器的初始⼤小，以及使⽤⼀个值填满你的容器。
但它也有⼀个std::initializer_list构造函数允许你使⽤花括号⾥⾯的值初始化容器。如果你创建⼀个数值类型的vector，然后你传递两个实参。把这两个实参放到小括号和放到花括号中是不同：
```cpp
std::vector<int> v1(10, 20); //使⽤⾮std::initializer_list
//构造函数创建⼀个包含10个元素的std::vector
//所有的元素的值都是20
std::vector<int> v2{10, 20}; //使⽤std::initializer_list
//构造函数创建包含两个元素的std::vector
//元素的值为10和20
```
- 第⼀，作为⼀个类库作者，你需要意识到如果你的⼀堆构造函数中重载过⼀个或者多个
std::initializer_list，⽤⼾代码如果使⽤了括号初始化，可能只会看到你重载的std::initializer_list这⼀个版本的构造函数。
- 第⼆，作为⼀个类库使⽤者，你必须认真的在花括号和小括号之间选择⼀个来创建对象。

如果你是⼀个模板的作者，花括号和小括号创建对象就更⿇烦了。通常不能知晓哪个会被使⽤。
举个例⼦，假如你想创建⼀个接受任意数量的参数，然后⽤它们创建⼀个对象。使⽤可变参数模板(variadic template )可以⾮常简单的解决：
```cpp
template<typename T, typename... Ts>
void doSomeWork(Ts&&... params) {
create local T object from params... …
}

T localObject(std::forward<Ts>(params)...); // 使⽤小括号
T localObject{std::forward<Ts>(params)...}; // 使⽤花括号

std::vector<int> v;
…
doSomeWork<std::vector<int>>(10, 20);
```
如果doSomeWork创建localObject时使⽤的是小括号，std::vector就会包含10个元素。
如果doSomeWork创建localObject时使⽤的是花括号，std::vector就会包含2个元素。
哪个是正确的？doSomeWork的作者不知道，只有调⽤者知道。
这正是标准库函数std::make_unique和std::make_shared（参⻅Item21）⾯对的问题。

## Item7-remember
- 括号初始化是最⼴泛使⽤的初始化语法，它防⽌变窄转换，并且对于C++最令⼈头疼的解析有天⽣的免疫性
- 在构造函数重载决议中，括号初始化尽最⼤可能与std::initializer_list参数匹配，即便其他构造函数看起来是更好的选择
- 对于数值类型的std::vector来说使⽤花括号初始化和小括号初始化会造成巨⼤的不同
- 在模板类选择使⽤小括号初始化或使⽤花括号初始化创建对象是⼀个挑战。