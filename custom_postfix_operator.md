# 自定义后缀操作符

C++11 新标准中引⼊了⽤户⾃定义字⾯量，也叫⾃定义后缀操作符，即通过实现⼀个后缀操作符，将申明了该后缀标识的字⾯
量转化为需要的类型。考察如下代码：

```cpp
long double operator"" _mm(long double x) { return x / 1000; }
long double operator"" _m(long double x) { return x; }
long double operator"" _km(long double x) { return x * 1000; }
int main()
{
cout << 1.0_mm << endl; //0.001
cout << 1.0_m << endl; //1
cout << 1.0_km << endl; //1000
return 0;
}
```

编译并运⾏：

```
0.001
1
1000
```

实际上，⾃定义字⾯量⼀般⽤于⽤户⾃定义的构造类型（结构体与类）。假如⼊我们有如下⼀个表⽰颜⾊的类。

```cpp
struct RGBA
{
    uint8_t r, g, b, a;
    RGBA(uint8_t r, uint8_t g, uint8_t b, uint8_t a):r(r),g(g),b(b),a(a){}
};

RGBA operator"" _RGBA(const char* str, size_t size)
{
    const char* r = nullptr, *g = nullptr, *b = nullptr, *a = nullptr;
    for (const char* p = str; p != str + size; ++p)
    {
        if (*p == 'r') r = p + 1;
        if (*p == 'g') g = p + 1;
        if (*p == 'b') b = p + 1;
        if (*p == 'a') a = p + 1;
    }
    if (r == nullptr || g == nullptr || b == nullptr) throw;
    if (a == nullptr)
    {
        return RGBA(atoi(r),atoi(g),atoi(b),0);
    }
    else
    {
        return RGBA(atoi(r), atoi(g), atoi(b),atoi(a));
    }
}
```

这⾥需要注意的是后缀操作符函数根据 C++ 11 标准，只有下⾯参数列表才是合法的：

```cpp
char const *
unsigned long long
long double
char const *, size_t
wchar_t const *, size_t
char16_t const *, size_t
char32_t const *, size_t
```

最后四个对于字符串相当有⽤，因为第⼆个参数会⾃动推断为字符串的长度。例如：

```cpp
size_t operator"" _len(char const * str, size_t size)
{
    return size;
}
int main()
{
    cout << "mike"_len <<endl; //结果为4
    return 0;
}
```

完成⾃定义后缀操作符函数后，我们可以使⽤⾃定义字⾯量来表⽰⼀个 RGBA 的对象了。

```cpp
//输出运算符重载
ostream& operator<<(ostream& os,const RGBA& color)
{
    return os<<"r="<< (int)color.r<<" g="<< (int)color.g<<" b="<< (int)color.b<<" a="<< (int)color.a<<endl;
}
int main()
{
    //⾃定义字⾯量来表⽰RGBA对象
    cout << "r255 g255 b255 a40"_RGBA << endl;
    return 0;
}
```

程序编译运⾏输出：

```
r=255 g=255 b=255 a=40
```
