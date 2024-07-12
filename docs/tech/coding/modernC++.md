## ModernC++ 学习笔记

参考来自https://changkun.de/modern-cpp/zh-cn

### 一、语言可用性

##### 常量

nullptr替换NULL，constexpr让用户显式的声明函数或对象构造函数在编译期会成为常量表达式

```cpp
static constexpr int lowbit(int x){
    return x & (-x);
}
```



**if/switch**：条件中可以有临时变量

```cpp
if (const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 3);
    itr != vec.end()) {
    *itr = 4;
}
```

##### 变量及初始化

**初始化列表**：`std::initializer_list`，用于大括号初始化对象和普通函数的形参

```cpp
//形参
void foo(std::initializer_list<int> list) {
    for (std::initializer_list<int>::iterator it = list.begin();
         it != list.end(); ++it) vec.push_back(*it);
}
//初始化对象 after C++11
class MagicFoo {
public:
    std::vector<int> vec;
    MagicFoo(std::initializer_list<int> list) {
        for (std::initializer_list<int>::iterator it = list.begin();
             it != list.end(); ++it)
            vec.push_back(*it);
    }
};
MagicFoo magicFoo = {1, 2, 3, 4, 5};
```

**结构化绑定**：可以让函数有多个返回值

```cpp
std::tuple<int, double, std::string> f() {
    return std::make_tuple(1, 2.3, "456");
}

int main() {
    auto [x, y, z] = f();
    std::cout << x << ", " << y << ", " << z << std::endl;
    return 0;
}
```

##### **类型推导**

+ auto，decltype，std::is_same<T,U>

auto不能用于函数形参进行类型推导，但是可以用于推导函数返回值

decltype(auto) 用于对转发函数或封装的返回类型进行推导，见下示例

```cpp
//常见用法
auto x = 1;
auto y = 2。0;
decltype(x+y) z;
if (std::is_same<decltype(z), int>::value)
    std::cout << "type z == int" << std::endl;
//auto推导函数返回值，尾返回类型推导
template<typename T, typename U>
auto add2(T x, U y) -> decltype(x+y){
    return x + y;
}
//after C++14
template<typename T, typename U>
auto add2(T x, U y){
    return x + y;
}
//decltype(auto)用法，推导封装函数返回值
decltype(auto) look_up_a_string_1() {
    return lookup1();
}
//该函数等同于
std::string look_up_a_string_1() {
    return lookup1();
}
```

##### 控制流

C++17之后，constexpr可以在if语句中，允许在代码中声明常量表达式的判断条件，如下使用则代码实际执行时表现为独立的两个print_type_info函数

```cpp
#include <iostream>

template<typename T>
auto print_type_info(const T& t) {
    if constexpr (std::is_integral<T>::value) {
        return t + 1;
    } else {
        return t + 0.001;
    }
}
int main() {
    std::cout << print_type_info(5) << std::endl;
    std::cout << print_type_info(3.14) << std::endl;
}
```

**区间for迭代**：注意带引用和不带引用的区别

```cpp
for (auto element : vec)
        std::cout << element << std::endl; // read only
for (auto &element : vec)
    element += 1;                     	 // writeable
```

##### 模板

作为C++的一大特色，这里需要着重记录一下

+ 外部模板：传统 C++ 中，模板只有在使用时才会被编译器实例化。换句话说，只要在每个编译单元（文件）中编译的代码中遇到了被完整定义的模板，都会实例化。通过外部模板可以告诉编译器何时进行实例化

```cpp
template class std::vector<bool>;          // 强行实例化
extern template class std::vector<double>; // 不在该当前编译文件中实例化模
```

+ using的使用：例如对函数指针，关于这个函数指针，代表的是它可以指向任意一个满足返回值是int，能够接收一个void *(即任何指针类型)参数的函数

```cpp
typedef int (*process)(void *)
using process = int(*)(void *) // 表示一个函数指针的别名
    
using foo = void(int); // 表示一个函数类型的别名，两者使用并无太大区别
```

+ 变长参数模板：允许任意个数，任意类别的模板参数，即代表0-N个

```cpp
template<typename R, typename... Ts> class Magic;

class Magic<int
    		std::vector<int>, 
	        std::map<std::string,std::vector<int>>
            > darkMagic
```

同样的**函数参数**也使用同样的表示代表不定长参数

```cpp
//定义
templace<typename... Args> void printf(const std::string &str, Args... )
//如何处理模板参数
//方法一：神奇的递归，遍历模板参数
#include <iostream>
template<typename T0>
void printf1(T0 value) {
    std::cout << value << std::endl;
}
template<typename T, typename... Ts>
void printf1(T value, Ts... args) {
    std::cout << value << std::endl;
    printf1(args...);
}
int main() {
    printf1(1, 2, "123", 1.1);
    return 0;
}
//方法二：变参模板展开，after C++17
templace<typename T0, typename... T>
void printf2(T0, t0, T... t){
    std::cout<<t0<<std::endl;
    if constexpr (sizeof...(t) > 0) printf2(t...)
}
```

折叠表达式：有点抽象，表达式也可以

```cpp
#include <iostream>
template<typename... T>
auto sum(T... t) {
    return (t + ...);
}
int main() {
    std::cout << sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) << std::endl;
}
```

+ 非类型模板参数：即字面量模板参数，在这个例子中让字面量10作为模板参数进行传递。甚至在C++17之后，能够使用auto

```cpp
//定义示例
template <typename T, int BufSize>
class buffer_t {
public:
    T& alloc();
    void free(T& item);
private:
    T data[BufSize];
}

buffer_t<int, 100> buf;

//auto帮助我们进行具体类型的推导
template <auto value> void foo() {
    std::cout << value << std::endl;
    return;
}

int main() {
    foo<10>();  // value 被推导为 int 类型
}
```

##### 面向对象

+ 委托构造：一个构造函数可以调用另一个构造函数

```cpp
#include <iostream>
class Base {
public:
    int value1;
    int value2;
    Base() {
        value1 = 1;
    }
    Base(int value) : Base() { // 委托 Base() 构造函数
        value2 = value;
    }
};

int main() {
    Base b(2);
    std::cout << b.value1 << std::endl;
    std::cout << b.value2 << std::endl;
}
```

+ 继承构造：子类通过using使得构造函数传递时不需要传递参数

```cpp
#include <iostream>
class Base {
public:
    int value1;
    int value2;
    Base() {
        value1 = 1;
    }
    Base(int value) : Base() { // 委托 Base() 构造函数
        value2 = value;
    }
};
class Subclass : public Base {
public:
    using Base::Base; // 继承构造
    Subclass(int value): Base(value){} //原始形式
};
int main() {
    Subclass s(3);
    std::cout << s.value1 << std::endl;
    std::cout << s.value2 << std::endl;
}
```

+ 显式虚函数重载：

当重载虚函数时，引入 `override` 关键字将显式的告知编译器进行重写

`final` 则是为了防止类被继续继承以及终止虚函数继续重写引入的。

```cpp
struct Base {
    virtual void foo(int);
};
struct SubClass: Base {
    virtual void foo(int) override; // 合法
    virtual void foo(float) override; // 非法, 父类没有此虚函数
};

struct Base {
    virtual void foo() final;
};

struct SubClass1 final: Base {
}; // 合法

struct SubClass2 : SubClass1 {
}; // 非法, SubClass1 已 final

struct SubClass3: Base {
    void foo(); // 非法, foo 已 final
};
```

+ 显式禁用默认函数：即禁用编译器为对象生成的默认构造函数，复制构造，赋值运算符以及析构函数。例如想让复制构造函数为private

```cpp
class Magic {
    public:
    Magic() = default; // 显式声明使用编译器生成的构造
    Magic& operator=(const Magic&) = delete; // 显式声明拒绝编译器生成构造
    Magic(int magic_number);
}
```

+ 强类型枚举：引入枚举类enum class
  + 传统C++中**同一个命名空间中的不同枚举类型的枚举值名字不能相同**
  + 类型安全，不会被隐式转换为整数
  + 不能够将其与整数数字进行比较， 不可能对不同的枚举类型的枚举值进行比较。但相同枚举值之间如果指定的值相同，那么可以进行比较：未指定枚举值类型默认为int

```cpp
//定义
enum class new_enum : unsigned int {
    value1,
    value2,
    value3 = 100,
    value4 = 100
};

//比较枚举值
if (new_enum::value3 == new_enum::value4) {
    // 会输出
    std::cout << "new_enum::value3 == new_enum::value4" << std::endl;
}

//重载<<，获得枚举值的值
#include <iostream>
template<typename T>
std::ostream& operator<<(
    typename std::enable_if<std::is_enum<T>::value,
        std::ostream>::type& stream, const T& e)
{
    return stream << static_cast<typename std::underlying_type<T>::type>(e);
}
```

### 二、语言运行期

##### Lambda表达式

捕获列表指的是传递外部数据，分为值捕获，引用捕获和隐式捕获。

+ 值捕获：被捕获的变量在 Lambda 表达式被创建时拷贝
+ 引用捕获：与引用传参类似，引用捕获保存的是引用，值会发生变化
+ 隐式捕获：可以在捕获列表中写一个 `&` 或 `=` 向编译器声明采用引用捕获或者值捕获，让编译器自行推导捕获列表

```cpp
[捕获列表](参数列表) mutable(可选) 异常属性 -> 返回类型 {
// 函数体
}

//示例
void lambda_reference_capture() {
    int value = 1;
    auto copy_value = [&value]{
        return value;
    }
    //等效于
    auto copy_value = [&]{
        return value;
    }
    value = 100;
    auto stored_value = copy_value();
    std::cout << "stored_value = " << stored_value << std::endl;
    // 这时, stored_value == 100, value == 100.
    // 因为 copy_value 保存的是引用
}
```

+ 表达式捕获：C++14之后，可以捕获右值(允许捕获的成员用任意的表达式进行初始化)

important 是一个独占指针，是不能够被 "=" 值捕获到，这时候我们可以将其转移为右值，在表达式中初始化。

```cpp
#include <memory>  // std::make_unique
#include <utility> // std::move

void lambda_expression_capture() {
    auto important = std::make_unique<int>(1);
    auto add = [v1 = 1, v2 = std::move(important)](int x, int y) -> int {
        return x+y+v1+(*v2);
    };
    std::cout << add(3,4) << std::endl;
}

```

+ 泛型Lambda：函数参数也可以用auto, after C++14

```cpp
auto add(auto x, auto y){
    return x + y;
}
```



##### 函数对象包装器

`std::function`：可调用对象的类型，它的实例可以对任何可以调用的目标实体进行存储、复制和调用操作。可以理解为函数的容器

Lambda 表达式的本质是一个和函数对象类型相似的类类型（称为闭包类型）的对象（称为闭包对象），捕获列表为空时退化为函数指针可进行传递

类型安全：C++中，类型安全意味着编译器能够检查类型匹配，防止不同类型之间的不合法操作，从而减少程序中的错误。举例：比如函数指针可以指向参数列表不匹配的函数，或者返回类型不匹配的函数，这样的操作是不安全的

```cpp
#include <iostream>

using foo = void(int); 
void functional(foo f) { 
    f(1); 
}

int main() {
    auto f = [](int value) {
        std::cout << value << std::endl;
    };
    functional(f); // 函数类型调用，传递闭包对象，隐式转换为 foo* 类型的函数指针值
    f(1); // lambda 表达式调用 
    
    //使用std::function
    int important = 10;
    std::function<int(int)> func = f;
    std::function<int(int)> func2 = [&](int value) -> int{
        return 1 + value + important;
    };
    return 0;
}
```

`std::bind, std::placeholder`：而 `std::bind` 则是用来绑定函数调用的参数的， 它解决的需求是我们有时候可能并不一定能够一次性获得调用某个函数的全部参数，通过这个函数， 我们可以将部分调用参数提前绑定到函数身上成为一个新的对象，然后在参数齐全后，完成调用

```cpp
int foo(int a, int b, int c) {
    ;
}
int main() {
    // 将参数1,2绑定到函数 foo 上，
    // 但使用 std::placeholders::_1 来对第一个参数进行占位
    auto bindFoo = std::bind(foo, std::placeholders::_1, 1,2);
    // 这时调用 bindFoo 时，只需要提供第一个参数即可
    bindFoo(1);
}
```

##### 右值引用

个人认为这也是C++11后最经典的让人困惑的问题，关于右值引用和通用引用，这里会做详细说明

+ 通用引用：`auto &&`，因为当auto被推到为不同的左右引用时，与 `&&` 的坍缩组合是完美转发

```cpp
//A simple Example:
template <typename Key, typename Value, typename F>
void update(std::map<Key, Value>& m, F foo) {
    for (auto&& [key, value] : m ) value = foo(key);
}
update(m, [](std::string key) -> long long int {
        return std::hash<std::string>{}(key);
});
```

左值(lvalue)：表达式结束后依然存在的持久对象

右值(rvalue)：表达式结束后就不再存在的临时对象

+ 纯右值(prvalue)：非引用返回的临时变量、运算表达式产生的临时变量、 原始字面量(除了字符串字面量)、Lambda 表达式都属于纯右值。
+ 但是数组可以被隐式转换为对应的指针类型，转换表达式的结果若不是左值引用，则一定是个右值（结果是右值引用为将亡值，否则为纯右值

```cpp
const char (&left)[6] = "01234"; // 正确，"01234" 类型为 const char [6]，因此是左值
const char*   p   = "01234";  // 正确，"01234" 被隐式转换为 const char*
const char*&& pr  = "01234";  // 正确，"01234" 被隐式转换为 const char*，该转换的结果是纯右值
```

+ 将亡值(xvalue)：临时的值能够被识别、同时又能够被移动。

  此处的左值 `temp` 会被进行隐式右值转换， 等价于 `static_cast<std::vector<int> &&>(temp)`，进而此处的 `v` 会将 `foo` 局部返回的值进行移动变成左值。

```cpp
std::vector<int> foo() {
    std::vector<int> temp = {1, 2, 3, 4};
    return temp;
}

std::vector<int> v = foo();
```

右值引用是什么：要拿到一个将亡值，就需要用到右值引用：`T &&`，只要变量还活着，将亡值就还活着

+ `std::move()`：用于将左值参数无条件转换为右值

```cpp
std::string lv1 = "string,"; // lv1 是一个左值
std::string&& rv1 = std::move(lv1); // 合法, std::move可以将左值转移为右值

const std::string& lv2 = lv1 + lv1; // 合法, 常量左值引用能够延长临时变量的生命周期
std::string& lv2 = lv1 + lv1; //非法，左值引用的初始值必须是一个左值

std::string&& rv2 = lv1 + lv2; // 合法, 右值引用延长临时对象生命周期
//虽然rv2引用了一个右值，但是它本身是个引用，所以是一个左值
```

移动语义：解决『移动』和『拷贝』两个概念的混淆问题

一个vector的例子

```cpp
int main() {
    std::string str = "Hello world.";
    std::vector<std::string> v;

    // 将使用 push_back(const T&), 即产生拷贝行为
    v.push_back(str);
    // 将输出 "str: Hello world."
    std::cout << "str: " << str << std::endl;

    // 将使用 push_back(const T&&), 不会出现拷贝行为
    // 而整个字符串会被移动到 vector 中，所以有时候 std::move 会用来减少拷贝出现的开销
    // 这步操作后, str 中的值会变为空
    v.push_back(std::move(str));
    // 将输出 "str: "
    std::cout << "str: " << str << std::endl;
    return 0;
}
```

一个对象的例子：

+ 解释：首先会在 `return_rvalue` 内部构造两个 `A` 对象，于是获得两个构造函数的输出。函数返回后，产生一个将亡值，被 `A` 的移动构造（`A(A&&)`）引用，从而延长生命周期，并将这个右值中的指针拿到，保存到了 `obj` 中，而将亡值的指针被设置为 `nullptr`，防止了这块内存区域被销毁。

```cpp
#include <iostream>
class A {
public:
    int *pointer;
    A():pointer(new int(1)) {
        std::cout << "构造" << pointer << std::endl;
    }
    A(A& a):pointer(new int(*a.pointer)) {
        std::cout << "拷贝" << pointer << std::endl;
    } // 无意义的对象拷贝
    A(A&& a):pointer(a.pointer) {
        a.pointer = nullptr;
        std::cout << "移动" << pointer << std::endl;
    }
    ~A(){
        std::cout << "析构" << pointer << std::endl;
        delete pointer;
    }
};
// 防止编译器优化
A return_rvalue(bool test) {
    A a,b;
    if(test) return a; // 等价于 static_cast<A&&>(a);
    else return b;     // 等价于 static_cast<A&&>(b);
}
int main() {
    A obj = return_rvalue(false);
    std::cout << "obj:" << std::endl;
    std::cout << obj.pointer << std::endl;
    std::cout << *obj.pointer << std::endl;
    return 0;
}
```

完美转发：一个声明的右值引用其实是一个左值。为了解决这个问题，需要`std::forward()`，可以保持原来的参数类型。

+ 引用坍缩规则：无论模板参数是什么类型的引用，当且仅当实参类型为右引用时，模板参数才能被推导为右引用类型

  <img src="../../assets/image-20240402215348307.png" alt="image-20240402215348307" style="zoom:50%;" />

+ 在如下的示例中：无论传递参数为左值还是右值，普通传参都会将参数作为左值进行转发，`std::move()`都会将传入的左值作为右值传递，`std::forward()`会保持引用类型

```cpp
void reference(int& v) {
    std::cout << "左值" << std::endl;
}
void reference(int&& v) {
    std::cout << "右值" << std::endl;
}

template <typename T>
void pass(T&& v) {
    std::cout << "普通传参:";
    reference(v); // 始终调用 reference(int&)
    
    std::cout << "       std::move 传参: ";
    reference(std::move(v));
    
    std::cout << "    std::forward 传参: ";
    reference(std::forward<T>(v));
    
    std::cout << "static_cast<T&&> 传参: ";
    reference(static_cast<T&&>(v));
}
int main() {
    std::cout << "传递右值:" << std::endl;
    pass(1); // 1是右值, 但输出是左值
    std::cout << "传递左值:" << std::endl;
    int l = 1;
    pass(l); // l 是左值, 输出左值
    return 0;
}
```



### 三、容器

##### 线性容器

`std::array`，与`vector`不同，`std::array`的大小是固定的，作用是让代码变得更加现代化，以及封装了一些操作函数

```cpp
std::array<int, 4> arr = {1, 2, 3, 4};

arr.empty(); // 检查容器是否为空
arr.size();  // 返回容纳的元素数
// 迭代器支持
for (auto &i : arr){}

// 用 lambda 表达式排序
std::sort(arr.begin(), arr.end(), [](int a, int b) {
    return b < a;
});

// 数组大小参数必须是常量表达式
constexpr int len = 4;
std::array<int, len> arr = {1, 2, 3, 4};

//C风格接口传参，注意无法隐式转换为退化的指针
void foo(int *p, int len) {
    return;
}

foo(&arr[0], arr.size());
foo(arr.data(), arr.size());
```

Tips: `vector`容器clear之后，需要调用`shrink_to_fit()`来清空内存

`std::forward_list`，和 `std::list` 的双向链表的实现不同，`std::forward_list` 使用单向链表进行实现， 提供了 `O(1)` 复杂度的元素插入，不支持快速随机访问（这也是链表的特点）， 也是标准库容器中唯一一个不提供 `size()` 方法的容器。当不需要双向迭代时，具有比 `std::list` 更高的空间利用率。



##### **无序容器**

现在常用的`std::unordered_map`/`std::unordered_multimap` 和`std::unordered_set`/`std::unordered_multiset`都是在C++11之后引入的，但是是无序的

`multiset`和`set`的区别：multiset允许容器有重复的元素。它可以看成一个序列，插入一个数， 删除一个数都能够在O(logn)的时间内完成，而且他能时刻保证序列中的数是有序的，而且序列中可以存在重复的数



##### **元组**

`std::tuple`，类似于Python，使用核心有三个函数`std::make_tuple, std::get, std::tie`

```cpp
auto get_student(int id){
    // 返回类型被推断为 std::tuple<double, char, std::string>
    return std::make_tuple(0,0, 'D', "null");
}
double gpa; char grade; std::string name;
//拆包
std::tie(gpa, grade, name) = get_student(1);
//get通过类型和位置获取，位置必须是一个常数
std::tuple<std::string, double, double, int> t("123", 4.5, 6.7, 8);
std::cout << std::get<std::string>(t) << std::endl;
std::cout << std::get<double>(t) << std::endl; // 非法, 引发编译期错误, 有两个double
std::cout << std::get<3>(t) << std::endl;
```

为了解决`std::get`依赖编译器常量的问题，C++17引入了`std::variant<T>`，从而容纳提供的几种类型的变量

+ 非常具有现代C++特色的tuple遍历

```cpp
#include <variant>
template <size_t n, typename... T>
constexpr std::variant<T...> _tuple_index(const std::tuple<T...>& tpl, size_t i) {
    if constexpr (n >= sizeof...(T))
        throw std::out_of_range("越界.");
    if (i == n)
        return std::variant<T...>{ std::in_place_index<n>, std::get<n>(tpl) };
    return _tuple_index<(n < sizeof...(T)-1 ? n+1 : 0)>(tpl, i);
}
template <typename... T>
constexpr std::variant<T...> tuple_index(const std::tuple<T...>& tpl, size_t i) {
    return _tuple_index<0>(tpl, i);
}
template <typename T0, typename ... Ts>
std::ostream & operator<< (std::ostream & s, std::variant<T0, Ts...> const & v) { 
    std::visit([&](auto && x){ s << x;}, v); 
    return s;
}

int i = 1;
std::cout << tuple_index(t, i) << std::endl;
```

元组合并和遍历：`std::tuple_cat`

```cpp
std::tuple<std::string, double, double, int> t("123", 4.5, 6.7, 8);
auto new_tuple = std::tuple_cat(std::make_tuple(0.0, 'D', "null"), std::move(t));

template <typename T>
auto tuple_len(T &tpl) {
    return std::tuple_size<T>::value;
}
for(int i = 0; i != tuple_len(new_tuple); ++i)
    std::cout << tuple_index(new_tuple, i) << std::endl;
```

### 四、智能指针和内存

**智能指针**：使用引用计数，对于动态分配的对象，进行引用计数，每当增加一次该对象的引用，计数就加一，当对象的引用计数为0时，就删除指向的堆内存。这种机制称为**RAII**

`std::shared_ptr, std::unique_ptr, std::weak_ptr`，需要头文件`<memory>`

##### std::shared_ptr

强引用指针，可以记录多个Ptr共同指向一个对象，可以消除显式的delete，不能直接new。但是有内存泄露的风险，考虑两个对象里有两个指针互相指向各自。

+ `get()`方法可以获取原始指针
+ `reset()`方法减少一个引用计数，`use_count()`查看一个对象的引用计数

```cpp
int main() {
    // auto pointer = new int(10); // illegal, no direct assignment
    // Constructed a std::shared_ptr
    auto pointer = std::make_shared<int>(10);
    auto pointer2 = pointer; // 引用计数+1
	auto pointer3 = pointer; // 引用计数+1
    int *p = pointer.get();  // 这样不会增加引用计数
    std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 3
    pointer2.reset();
    std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl;   // 2
    std::cout << "pointer2.use_count() = "
          << pointer2.use_count() << std::endl;           // pointer2 已 reset; 0
    std::cout << *pointer << std::endl; // 10
    // The shared_ptr will be destructed before leaving the scope
    return 0;
}
```

##### std::unique_ptr：

独占指针，禁止其他智能指针与其共享一个对象，不可复制，但是可以用`std::move`转移给其他指针

```cpp
std::unique_ptr<int> pointer = std::make_unique<int>(10); //after C++14
std::unique_ptr<int> pointer2 = pointer; // 非法

template<typename T, typename ...Args>
std::unique_ptr<T> make_unique( Args&& ...args ) {
  return std::unique_ptr<T>( new T( std::forward<Args>(args)... ) );
}

std::unique_ptr<Foo> p1(std::make_unique<Foo>());
std::unique_ptr<Foo> p2(std::move(p1));
```

##### std::weak_ptr：

弱引用指针，弱引用不会引起引用计数增加。但是也不能对资源进行操作

> `std::weak_ptr` 没有 `*` 运算符和 `->` 运算符，所以不能够对资源进行操作
>
> 它可以用于检查 `std::shared_ptr` 是否存在，其 `expired()` 方法能在资源未被释放时，会返回 `false`，否则返回 `true`；
>
> 除此之外，它也可以用于获取指向原始对象的 `std::shared_ptr` 指针，其 `lock()` 方法在原始对象未被释放时，返回一个指向原始对象的 `std::shared_ptr` 指针，进而访问原始对象的资源，否则返回`nullptr`。

### 五、正则表达式

##### 正则表达式

分为特殊字符和限定符，将他们组合在一起就是正则。中括号表达式就是作为一个整体

<img src="../../assets/image-20240407184014235.png" alt="image-20240407184014235" style="zoom:67%;" />

<img src="../../assets/image-20240407184023237.png" alt="image-20240407184023237" style="zoom:67%;" />

##### std::regex

操作`std::string`对象，利用模式`std::regex`，通过`std::regex_match`进行匹配，从而差生`std::smatch`

+ `std::smatch`存放了匹配的结果

```cpp
std::string fnames[] = {"foo.txt", "bar.txt", "test", "a0.txt", "AAA.txt"};
// 为使 \. 作为正则表达式传递进去生效，需要对 \ 进行二次转义，从而有 \\.
std::regex txt_regex("[a-z]+\\.txt");
for (const auto &fname: fnames)
    std::cout << fname << ": " << std::regex_match(fname, txt_regex) << std::endl;
}

std::regex base_regex("([a-z]+)\\.txt");
std::smatch base_match;
for(const auto &fname: fnames) {
    if (std::regex_match(fname, base_match, base_regex)) {
        // std::smatch 的第一个元素匹配整个字符串
        // std::smatch 的第二个元素匹配了第一个括号表达式
        if (base_match.size() == 2) {
            std::string base = base_match[1].str();
            std::cout << "sub-match[0]: " << base_match[0].str() << std::endl;
            std::cout << fname << " sub-match[1]: " << base << std::endl;
        }
    }
}
```



### 六、并行与并发

##### std::thread

基于`std::thread`的一套api，示例如下

+ `get_id()`获取所创建线程的线程ID，`join()`来等待一个线程结束

```cpp
#include<thread>

int main() {
    std::thread t([](){
        std::cout << "hello world." << std::endl;
    });
    t.join();
    return 0;
}
```

##### std::mutex

+ `lock()`上锁，`unlock()`解锁，调用`lock()`后需要在每个临界区的出口处调用`unlock()`
+ RAII语法，模板类`std::lock_guard`，临界区的互斥量的创建只需要在作用域的开始部分

```cpp
#include<mutex>
#include<thread>

int v = 1;

void critical_section(int change_v) {
    static std::mutex mtx;
    std::lock_guard<std::mutex> lock(mtx);

    // 执行竞争操作
    v = change_v;

    // 离开此作用域后 mtx 会被释放
}

int main(){
    std::thread t1(critical_section, 3), t2(critical_section, 3);
    t1.join();
    t2.join();
}

```

+ `std::unique_lock`：以独占所有权的方式管理mutex对象上的上锁和解锁的操作，和`std::lock_guard`相比，可以在声明后任意位置显示调用`lock`和`unlock`，从而控制锁的作用范围。`std::condition_variable::wait`必须要用`std::unique_lock`作为参数

```cpp
void critical_section(int change_v){
    static std::mutex mtx;
    std::unique_lock<std::mutex> lock(mtx);
    v = change_v;
    std::cout << v << std::endl;
    // 将锁进行释放
    lock.unlock();

    // 在此期间，任何人都可以抢夺 v 的持有权

    // 开始另一组竞争操作，再次加锁
    lock.lock();
    v += 1;
    std::cout << v << std::endl;      
}
```

##### std::future

用于获取异步任务的结果。

+ `std::packaged_task`：用于封装任何可以调用的目标，从而用于实现异步的调用
+ 封装好调用目标后，`get_future()`可以获得一个`std::future()`对象以便用于实施线程同步

```cpp
#include<future>
#include<thread>

int main(){
    std::packaged_task<int()> task([]({return 7;}));
    std::future<int> result = task.get_future(); // 在一个线程中执行 task
    std::thread(std::move(task)).detach();
    std::cout << "waiting...";
    result.wait(); // 在此设置屏障，阻塞到期物的完成
    std::cout << "done!" << std:: endl << "future result is "
              << result.get() << std::endl;
}
```

`std::condition_variable`

+ 条件变量用于解决死锁
+ `notify_one()`用于唤醒一个线程，`notify_all()`通知所有线程

```cpp
#include <condition_variable>
#include <chrono>
int main(){
    std::queue<int> produced_nums;
    std::mutex mtx;
    std::condition_variable cv;
    bool notified = false;
    
    auto producer = [&](){
        for(int i = 0; ;i++){
            std::this_thread::sleep_for(std::chrono::milliseconds(900));
            std::unique_lock<std::mutex> lock(mtx);
            std::cout<< "producing "<<i<<std::endl;
            produced_nums.push(i);
            notified = true;
            cv.notified_all();
        }
    };
    
    auto consumer = [&](){
        while(true){
             std::unique_lock<std::mutex> lock(mtx);
            while(!notified){ // 避免虚假唤醒
                cv.wait(lock);
            }
            // 短暂取消锁，使得生产者有机会在消费者消费空前继续生产
            lock.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
            lock.lock();
            while (!produced_nums.empty()) {
                std::cout << "consuming " << produced_nums.front() << std::endl;
                produced_nums.pop();
            }
            notified = false;            		 
        }
    }
    
    std::thread p(producer);
    std::thread cs[2];
    for (int i = 0; i < 2; ++i) {
        cs[i] = std::thread(consumer);
    }
    p.join();
    for (int i = 0; i < 2; ++i) {
        cs[i].join();
    }
}
```

##### 原子操作和内存模型

以guide中的例子举例：由于a和flag两个变量在并行的线程中被读写，出现了竞争，以及编译器对指令重排的影响造成b的最终结果不确定

```cpp
int main() {
    int a = 0;
    int flag = 0;

    std::thread t1([&]() {
        while (flag != 1);

        int b = a;
        std::cout << "b = " << b << std::endl;
    });

    std::thread t2([&]() {
        a = 5;
        flag = 1;
    });

    t1.join();
    t2.join();
    return 0;
}
```

`std::mutex()`太强了，对于那些没有中间状态的变量来说，可以使用`std::atomic`来针对共享变量的读写，将原子类型的读写从一组CPU指令最小化到单个CPU指令

+ `fetch_add, fetchsub`实现基本的加减，基本类型已经重载进了运算符
+ `std::atomic<T>::is_lock_free`检查某个原子类型是否支持原子操作

```cpp
std::atomic<int> count = {0};
std::thread t1([](){
    count.fetch_add(1);
});
std::thread t2([](){
    count++;        // 等价于 fetch_add
    count += 1;     // 等价于 fetch_add
});

struct A{float x; int y; long long z;};

std::atomic<A> a;
std::cout << std::boolalpha << a.is_lock_free() << std::endl;
```

+ 一致性模型：

  + 线性一致性：要求任何一次读操作都能读到某个数据的最近一次写的数据，并且所有线程的操作顺序与全局时钟下的顺序是一致的
  + 顺序一致性：要求任何一次读操作都能读到某个数据的最近一次写的数据。
  + 因果一致性：只需要有因果关系的操作顺序得到保障
  + 最终一致性：保障某个操作在未来的某个时间节点上会被观察到，但并未要求被观察到的时间

+ 内存模型：C++为原子操作定义的六种内存顺序`std::memory_order`，可以体现四种同步模型

  + `std::memory_order_relaxed`，单个线程内的原子操作都是顺序执行的，不允许指令重排，但不同线程间原子操作的顺序是任意的
  + `std::memory_order_consume`，限制进程间的操作顺序

  ```cpp
  // 初始化为 nullptr 防止 consumer 线程从野指针进行读取
  std::atomic<int*> ptr(nullptr);
  int v;
  std::thread producer([&]() {
      int* p = new int(42);
      v = 1024;
      ptr.store(p, std::memory_order_release);
  });
  std::thread consumer([&]() {
      int* p;
      while(!(p = ptr.load(std::memory_order_consume)));
  
      std::cout << "p: " << *p << std::endl;
      std::cout << "v: " << v << std::endl;
  });
  producer.join();
  consumer.join();
  ```

  + `std::memory_order_release`和`std::memory_order_acquire`：进一步加紧对不同线程间原子操作的顺序的限制。release之前的所有写操作，对其他线程的任何acquire操作都是可见的。

  > `std::memory_order_release` 确保了它之前的写操作不会发生在释放操作之后，是一个向后的屏障（backward），而 `std::memory_order_acquire` 确保了它之前的写行为不会发生在该获取操作之后，是一个向前的屏障（forward）。对于选项 `std::memory_order_acq_rel` 而言，则结合了这两者的特点，唯一确定了一个内存屏障，使得当前线程对内存的读写不会被重排并越过此操作的前后

  `compare_exchange_strong(compare-and-swap)`:比较交换原语

  ```cpp
  std::vector<int> v;
  std::atomic<int> flag = {0};
  std::thread release([&]() {
      v.push_back(42);
      flag.store(1, std::memory_order_release);
  });
  std::thread acqrel([&]() {
      int expected = 1; // must before compare_exchange_strong
      while(!flag.compare_exchange_strong(expected, 2, std::memory_order_acq_rel))
          expected = 1; // must after compare_exchange_strong
      // flag has changed to 2
  });
  std::thread acquire([&]() {
      while(flag.load(std::memory_order_acquire) < 2);
  
      std::cout << v.at(0) << std::endl; // must be 42
  });
  release.join();
  acqrel.join();
  acquire.join();
  ```

  + `std::memory_order_seq_cst`：原子操作满足顺序一致性，损耗性能

### 七、其他杂项

##### std::chrono

一个time library，用于处理时间的模板库，包括`duration`、`time_point`、`clock`三个概念

`duration`:  表示一段时间

+ `Rep`表示Period的数量，`Period`用来表示用秒表示的时间单位

+ `template <intmax_t N, intmax_t D = 1> class ratio`：N是分子，D是分母
+ `duration`之间的转换使用`std::chrono::duration_cast<type>(variable)`,成员函数`count()`返回Rep类型的Period数量

```cpp
template <class Rep, class Period = ratio<1> > class duration;
ratio<3600, 1> //代表1 hour
```

```cpp
#include<chrono>
#include<ratio>

int main(){
    typedef std::chrono::duration<int> seconds_type;
    typedef std::chrono::duration<int,std::milli> milliseconds_type;
    typedef std::chrono::duration<int,std::ratio<60*60>> hours_type;
    
    hours_type h_oneday (24);                  // 24h
    seconds_type s_oneday (60*60*24);
    seconds_type s_onehour (60*60); 
    hours_type h_onehour (std::chrono::duration_cast<hours_type>(s_onehour)); //类型转换
    std::milliseconds foo(1000);   //1s
    foo *= 60;
    
    std::cout << foo.count() * milliseconds::period::num / milliseconds::period::den; //60000 * 1 / 1000即Ratio的分子与分母
    std::cout << " seconds.\n"; 
    
}
```

`Time point`：表示一个具体时间

+ `template <class Clock, class Duration = typename Clock::duration>  class time_point;`
+ `time_from_eproch()`用来获得1970年1月1日到time_point时间经过的duration

```cpp
using namespace std::chrono;
  
time_point <system_clock,duration<int>> tp_seconds (duration<int>(1));

system_clock::time_point tp (tp_seconds);

std::cout << "1 second since system_clock epoch = ";
std::cout << tp.time_since_epoch().count();
std::cout << " system_clock periods." << std::endl;

// display time_point:
std::time_t tt = system_clock::to_time_t(tp);
std::cout << "time_point tp is: " << ctime(&tt);
```

`Clocks`: 表示当时钟，最常用

+ `std::chrono::system_clock`表示当前的系统时钟，系统中运行的所有进程使用now()得到的时间是一致的

+ `now()`、`to_time_t()`、`from_time_t()`

```cpp
using std::chrono::system_clock;

std::chrono::duration<int,std::ratio<60*60*24> > one_day (1);
system_clock::time_point today = system_clock::now()
system_clock::time_point tomorrow = today + one_day;

std::time_t tt;
  
tt = system_clock::to_time_t ( today );
std::cout << "today is: " << ctime(&tt);    
```

+ `std::chrono::steady_clock`: 表示稳定的时间间隔，即中途修改了系统时间，也不影响`now()`的结果

```cpp
using namespace std::chrono;

steady_clock::time_point t1 = steady_clock::now();
  
std::cout << "printing out 1000 stars...\n";
for (int i=0; i<1000; ++i) std::cout << "*";
std::cout << std::endl;

steady_clock::time_point t2 = steady_clock::now();

duration<double> time_span = duration_cast<duration<double>>(t2 - t1);

std::cout << "It took me " << time_span.count() << " seconds.";
std::cout << std::endl;
```

##### std::filesystem

方便的，跨平台的与文件系统交互

核心特性：

+ path object：路径对象
+ directory_entry：文件入口
+ directory iterators：获取文件系统目录中文件的迭代器容器，元素为`directory_entry`对象
+ file status: 用于获取和修改文件（或目录）的属性
+ supportive functions
  + getting information about the path
  + files manipulation: copy, move, create, symlinks
  + last write time
  + permissions
  + space/filesize

使用实例：

+ 检查路径存在，根路径名，根目录路径，与根目录的相对路径，具体文件的父目录路径、文件名、文件名的stem和扩展名

```cpp
namespace fs = std::experimental::filesystem

fs::path Path(""C:\\Windows\\system.ini"");
cout << "exists() = " << fs::exists(pathToShow) << "\n"
     << "root_name() = " << pathToShow.root_name() << "\n"
     << "root_path() = " << pathToShow.root_path() << "\n"
     << "relative_path() = " << pathToShow.relative_path() << "\n"
     << "parent_path() = " << pathToShow.parent_path() << "\n"
     << "filename() = " << pathToShow.filename() << "\n"
     << "stem() = " << pathToShow.stem() << "\n"
     << "extension() = " << pathToShow.extension() << "\n";
```

+ 遍历一个Path的每个部分：

```cpp
int i = 0;
for (const auto& part : Path)
    cout << "path part: " << i++ << " = " << part << "\n";
```

+ 添加Path字符串：

```cpp
fs::path p1("C:\\temp");
p1 /= "user"; //等效于p1 += "\\user", 会产生C:\\temp\\user
```

+ 获取文件大小：
  + 检查了边界值，`static_cast<uintmax_t>(-1)`是最大可能整数值的无符号整数。

```cpp
uintmax_t ComputeFileSize(const fs:path& Path){
    if(fs::exists(Path) && fs::is_regular_file(Path)){
        	auto err = std::error_code();
        	auto filesize = fs::file_size(Path, err);
        	if(filesize != static_cast<uintmax_t>(-1)){
                return filesize;
            }
    }
    return static_cast<uintmax_t>(-1);
}
```

+ 获取最后修改时间：
  + `std::localtime()` 函数将 `time_t` 类型的时间转换为本地时间的结构体表示，并返回一个指向该结构体的指针。
  + `std::asctime()` 函数将该结构体指针作为参数，将其格式化为字符串形式

```cpp
auto timeEntry = fs::last_write_time(entry);
time_t cftime = chrono::system_clock::to_time_t(timeEntry);
cout << std::asctime(std::localtime(&cftime));
```

+ 遍历目录：

```cpp
void DisplayDirTree(const fs::path& pathToShow, int level)
{
    if (fs::exists(pathToShow) && fs::is_directory(pathToShow))
    {
        auto lead = std::string(level * 3, ' ');
        for (const auto& entry : fs::directory_iterator(pathToShow))
        {
            auto filename = entry.path().filename();
            if (fs::is_directory(entry.status()))
            {
                cout << lead << "[+] " << filename << "\n";
                DisplayDirTree(entry, level + 1);
                cout << "\n";
            }
            else if (fs::is_regular_file(entry.status()))
                DisplayFileInfo(entry, lead, filename);
            else
                cout << lead << " [?]" << filename << "\n";
        }
    }
}
```



##### noexcept

+ 用于修饰函数不可能抛出异常，如果抛出异常，编译器会使用`std::terminate()`来终止程序运行
+ 修饰表达式，表达式无异常时返回`true`
+ 封锁异常扩散，如果内部产生异常，外部也不会触发
+ 可以让编译器优化代码

`std::boolalhpa`：让bool值以true或false的形式输出，而不是1和0

```cpp
void no_throw() noexcept{
    return;
}

auto block_throw = []() noexcept{
    no_throw();
};

std::cout<<std::boolalpha<< "lno_throw() noexcept? " << noexcept(block_throw()) << std::endl;

```

##### 原始字符串字面量

不需要使用`\`来修饰特殊符号

```cpp
std::string str = R"(C:\File\To\Path)";
std::cout << str << std::endl;
```

自定义字面量：通过重载`""`运算符来实现

```cpp
std::string operator"" _wow1(const char *wow1, size_t len) {
    return std::string(wow1)+"woooooooooow, amazing";
}

std::string operator"" _wow2 (unsigned long long i) {
    return std::to_string(i)+"woooooooooow, amazing";
}

auto str = "abc"_wow1;
auto num = 1_wow2;
```

##### 内存对齐

![image-20240417084714528](../../assets/image-20240417084714528.png)

`alignof、alignas`: 用于对内存对齐进行控制，前者能够获得与平台相关的`std::size_t`的值，后者可以重新修饰某个结构的定义方式

`std::max_align_t`：要求每个标量类型的对齐方式一样并为最大标量，大部分平台都是long double

```cpp
#include<iostream>

struct Storage {
    char      a;
    int       b;
    double    c;
    long long d;
};

struct alignas(std::max_align_t) AlignasStorage {
    char      a;
    int       b;
    double    c;
    long long d;
};

int main() {
    std::cout << alignof(Storage) << std::endl;
    std::cout << alignof(AlignasStorage) << std::endl;
    return 0;
}
```

