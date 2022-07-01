---
layout: post
title: C++ 移动语义
date: 2022-04-15 20:30:32 +0800
categories: C++
---

C++中基于值语义的拷贝和赋值严重影响了程序性能。编译器通过使用RVO、NRVO以及复制省略技术，尽量减小拷贝次数来提升代码的运行效率。自C++11起，引入了移动语义，实现了在保留原有的语法并不改动已存在的代码的基础上提升代码运行效率的目的。

#### 值语义
值语义(value semantics)指目标对象由源对象拷贝生成，且生成后与源对象完全无关，彼此独立存在，改变互不影响，就像int类型互相拷贝一样。C++的内置类型(bool/int/double/char)都是值语义，标准库里的complex<> 、pair<>、vector<>、map<>、string等等类型也都是值语意，拷贝之后就与原对象脱离关系。

C++中基于值语义的拷贝构造和赋值拷贝，会招致对资源密集型对象不必要拷贝，大量的拷贝很可能成为程序的性能瓶颈。

#### 左值和右值

##### 左值
左值是表达式结束后依然存在的对象，比如：赋值符号左边的值。可以将左值看作是一个关联了名称的内存位置，允许程序的其他部分来访问它。在这里，我们将 "名称" 解释为任何可用于访问内存位置的表达式。所以，如果 arr 是一个数组，那么 arr[1] 和 *(arr+1) 都将被视为相同内存位置的“名称”。

左值具有以下特征：
* 可通过取地址运算符获取其地址
* 可修改的左值可用作内建赋值和内建符合赋值运算符的左操作数
* 可以用来初始化左值引用

哪些都是左值呢?主要包含以下类型：
* 变量名、函数名以及数据成员名
* 返回左值引用的函数调用
* 由赋值运算符或复合赋值运算符连接的表达式，如a=b, a-=b等
* 解引用表达式*ptr
* 前置自增和自减表达式，如++a, ++b
* 成员访问运算符（"."运算符）的结果
* 通过指针访问成员（ -> ）运算符的结果
* 下标运算符的结果([])
* 字符串字面值，如"abc"。C++将字符串字面值实现为char型数组，存储在常量区，实实在在地为每个字符都分配了空间并且允许程序员对其进行操作，因此可以对字符串字面值取地址，例如：std::cout << &"abc" << std::endl。

注：对于一个表达式，凡是对其取地址（&）操作可以成功的都是左值。
例如：
```
int a = 1; // a是左值
T& f();
f();//左值
++a;//左值
--a;//左值
int b = a;//a和b都是左值
struct S* ptr = &obj; // ptr为左值
arr[1] = 2; // 左值
int *p = &a; // p为左值
*p = 10; // *p为左值
class MyClass{};
MyClass c; // c为左值
"abc"
```

##### 右值
C++11将右值分为纯右值和将亡值两种。

（1）纯右值

纯右值就是C++11标准之前的右值的概念。比如：非引用返回的函数返回的临时变量值；一些运算表达式（如1+2产生的临时结果）；不跟对象关联的字面量值（如2，'c'，true）。对这些值都不能够取地址。

纯右值具有以下特征：
* 不会是多态
* 不会是抽象类型或数组
* 不会是不完全类型

以下表达式的值都是纯右值：
* 字面值(字符串字面值除外)，例如1，'a', true等
* 返回值为非引用的函数调用或操作符重载，例如：str.substr(1, 2), str1 + str2, or it++
* 后置自增和自减表达式(a++, a--)
* 算术表达式
* 逻辑表达式
* 比较表达式
* 取地址表达式
* lambda表达式

例如：
```
nullptr;
true;
1;
int fun();
fun();

int a = 1;
int b = 2;
a + b;

a++;
b--;

a > b;
a && b;
```

（2）将亡值

将亡值是C++11新增的和右值引用相关的表达式，这样的表达式通常是将要被移动的对象、返回右值引用（T&&）的函数返回值、std::move()函数的返回值等。将亡值是具名的临时值，同时又能够被移动。

将亡值只能通过两种方式来获得：
* 返回右值引用的函数的调用表达式,如 static_cast<T&&>(t); 
* 转换为右值引用的转换函数的调用表达式，如：std::move(t)

#### 左值引用和右值引用
对左值的引用称为左值引用，而对右值的引用称为右值引用。在C++11中，左值引用与C++11之前的引用含义相同，用&来表示，为了进行区分，右值引用用&&来表示。右值的生命周期很短，右值引用的引入，使得可以延长右值的生命周期。

```
int a = 1;
int &b = a;
int &c = 10; // 错，10是一个常量，也就是右值，一个右值不能够被左值引用绑定
```
在C++11之前，我们通过会说b是对a的一个引用，但是在C++11之后更为精确的说法是b是一个左值引用。

```
int &&a = 1;  // a是一个右值引用
int b = 1;
int &&c = b; // 错误，右值引用不能绑定左值
```
跟左值引用一样，右值引用不会发生拷贝，并且右值引用等号右边必须是右值，如果是左值则会编译出错，当然这里也可以进行强制转换，这将在后面提到。

注意：绑定右值的右值引用，其变量本身是个左值。为了便于理解，代码如下：
```
#include <iostream>

int fun(int &a) {
    std::cout << "in fun(int &)" << std::endl;
}

int fun(int &&a) {
    std::cout << "in fun(int &&)" << std::endl;
}

int main() {
    int a = 1;
    int &&b = 1;

    fun(a);
    fun(b);
    fun(1);

    return 0;
}
```
代码输出如下：
```
in fun(int &)
in fun(int &)
in fun(int &&)
```
一个表达式有两个属性，分别为类型和值类别。左值引用和右值引用就属于类型,而左值和右值则属于值类别范畴。

左值引用和右值引用的规则如下:
* 左值引用，使用T&，只能绑定左值
* 右值引用，使用T&&，只能绑定右值
* 常量左值引用，使用const T&，既可以绑定左值，又可以绑定右值，但是不能对其进行修改
* 具名右值引用，编译器会认为是个左值
* s编译器的优化需要满足特定条件，不能过度依赖

例如：
```
int a = 1;
int &rb = a; // 正确，rb为左值引用
int &&rrb = a; // 错误，a是左值，右值引用不能绑定左值
int &&rrb1 = 1; // 正确，1为右值
int &rb1 = i * 2; // 错误，i * 2是右值，而rb1为左值引用
int &&rrb2 = i * 2; // 正确
const int &c = 1; // 正确
const int &c1 = i * 2; // 正确
```

除了可以根据规则区分左值引用和右值引用，系统也提供了API可以进行判断：std::is_lvalue_reference和std::is_rvalue_reference。

例如：
```
int a = 1;
int &ra = a;
int &&b = 1;

std::cout << std::is_lvalue_reference<decltype(ra)>::value << std::endl;
std::cout << std::is_rvalue_reference<decltype(ra)>::value << std::endl;
std::cout << std::is_rvalue_reference<decltype(b)>::value << std::endl;
```
运行结果：
```
1
0
1
```

#### 移动语义
自C++11引入右值引用后，可以结合移动语义通过延长临时变量的生命周期避免拷贝，进而达到性能优化的目的。在C++11之前，当进行值传递时，编译器会隐式调用拷贝构造函数；自C++11起，通过右值引用来避免由于拷贝调用而导致的性能损失。

右值引用的主要用途是创建移动构造函数和移动赋值运算符。移动构造函数和拷贝构造函数一样，将对象的实例作为其参数，并从原始对象创建一个新的实例。但是，移动构造函数可以避免内存重新分配，移动构造函数的参数是一个右值引用，也可以说是一个临时对象，而临时对象在调用之后就会被销毁不再被使用，因此，在移动构造函数中对参数进行移动而不是拷贝。换句话说，右值引用和移动语义允许我们在使用临时对象时避免不必要的拷贝。

移动语义通过移动构造函数和移动赋值操作符实现，移动构造函数和移动赋值操作符需要满足如下条件：
* 参数对象的类型必须为右值引用，即 T&&
* 参数对象不可以是常量，因为函数内部需要修改对象
* 参数对象的成员转移后需要修改（如改为nullptr），避免临时对象的析构函数将资源释放掉

```
#include <iostream>
#include <vector>

class BigObj {
public:
    explicit BigObj(size_t length)
            : length_(length), data_(new int[length]) {
        std::cout << "BigObj(size_t length)" << std::endl;
    }

    // Destructor.
    ~BigObj() {
        if (data_ != NULL) {
            delete[] data_;
            length_ = 0;
            std::cout << "~BigObj()" << std::endl;
        }
    }

    // 拷贝构造函数
    BigObj(const BigObj& other)
            : length_(other.length_), data_(new int[other.length_]) {
        std::copy(other.data_, other.data_ + length_, data_);
        std::cout << "BigObj(const BigObj& other)" << std::endl;
    }

    // 赋值运算符
    BigObj& operator=(const BigObj& other) {
        if (this != &other) {
            delete[] data_;
            length_ = other.length_;
            data_ = new int[length_];
            std::copy(other.data_, other.data_ + length_, data_);
        }
        std::cout << "BigObj& operator=(const BigObj& other)" << std::endl;
        return *this;
    }

    // 移动构造函数
    BigObj(BigObj&& other) : data_(nullptr), length_(0) {
        data_ = other.data_;
        length_ = other.length_;

        other.data_ = nullptr;
        other.length_ = 0;
        std::cout << "BigObj(BigObj&& other)" << std::endl;
    }

    // 移动赋值运算符
    BigObj& operator=(BigObj&& other) {
        if (this != &other) {
            delete[] data_;

            data_ = other.data_;
            length_ = other.length_;

            other.data_ = NULL;
            other.length_ = 0;
        }
        std::cout << "BigObj& operator=(BigObj&& other)" << std::endl;
        return *this;
    }

private:
    size_t length_;
    int* data_;
};

int main() {
    std::vector<BigObj> v;
    std::cout << "----------------------------------" << std::endl;
    v.push_back(BigObj(25));
    std::cout << "----------------------------------" << std::endl;
    v.push_back(BigObj(75));
    std::cout << "----------------------------------" << std::endl;

    v.insert(v.begin() + 1, BigObj(50));
    std::cout << "----------------------------------" << std::endl;
    return 0;
}
```
运行结果：
```
----------------------------------
BigObj(size_t length)
BigObj(BigObj&& other)
----------------------------------
BigObj(size_t length)
BigObj(BigObj&& other)
BigObj(const BigObj& other)
~BigObj()
----------------------------------
BigObj(size_t length)
BigObj(BigObj&& other)
BigObj(const BigObj& other)
BigObj(const BigObj& other)
~BigObj()
~BigObj()
----------------------------------
~BigObj()
~BigObj()
~BigObj()

```

##### 移动构造函数
移动构造函数定义如下：
```
BigObj(BigObj&& other) : data_(nullptr), length_(0) {
    data_ = other.data_;
    length_ = other.length_;

    other.data_ = nullptr;
    other.length_ = 0;
}
```
从上述代码可以看出，移动构造函数不分配任何新资源，也不会复制其它资源：参数对象other中的内存被移动到新成员后，other中原有的内容则消失了。换句话说，它窃取了other的资源，然后将other设置为其默认构造的状态。在移动构造函数中，最最关键的一点是，它没有额外的资源分配，仅仅是将其它对象的资源进行了移动，占为己用。

在此，我们假设data_很大，包含了数百万个元素。如果使用原来拷贝构造函数的话，就需要将该数百万元素挨个进行复制，性能可想而知。而如果使用该移动构造函数，因为不涉及到新资源的创建，不仅可以节省很多资源，而且性能也有很大的提升。

##### 移动赋值运算符
移动赋值运算符定义如下：
```
BigObj& operator=(BigObj&& other) {
    if (this != &other) {
        delete[] data_;

        data_ = other.data_;
        length_ = other.length_;

        other.data_ = NULL;
        other.length_ = 0;
    }
    return *this;
}
```
移动赋值运算符的操作步骤如下：释放当前拥有的资源；窃取他人资源；将他人资源设置为默认状态；返回*this。

在定义移动赋值运算符的时候，需要进行判断被移动的对象是否跟目标对象一致，否则可能会出问题，如下代码：
```
bigObj = std::move(bigObj);
```
源和目标是同一个对象，这可能会导致一个严重的问题：它最终可能会释放它试图移动的资源。为了避免此问题，我们需要通过判断来进行，比如可以如下操作：
```
if (this == &other) {
  return *this
}
```

##### 生成时机
在C++中有四个特殊的成员函数：默认构造函数、析构函数，拷贝构造函数，拷贝赋值运算符，之所以称之为特殊的成员函数，这是因为如果开发人员没有定义这四个成员函数，编译器也会在满足某些特定条件(仅在需要的时候才生成，比如某个代码使用它们但是它们没有在类中明确声明)下，自动生成。这些由编译器生成的特殊成员函数是public且inline。C++11起又引入了另外两只特殊的成员函数：移动构造函数和移动赋值运算符。如果开发人员没有显示定义移动构造函数和移动赋值运算符，那么编译器也会生成默认。

与其他四个特殊成员函数不同，编译器生成默认的移动构造函数和移动赋值运算符需要满足以下规则：
* 只有类中没有显式定义拷贝构造函数、赋值运算符以及析构函数，且类的每个非静态成员都可以移动时，编译器才会生成默认的移动构造函数或者移动赋值运算符。如果类中显式定义了拷贝构造函数、拷贝赋值运算符或者析构函数(这三者之一，表示程序员要自己处理对象的复制或释放问题)，编译器就不会为生成默认的移动构造函数或者移动赋值运算符，这样做的目的是防止编译器生成的默认移动构造函数或者移动赋值运算符不是开发人员想要的。
* 如果类中显式声明了移动构造函数或移动赋值运算符，则拷贝构造函数和拷贝赋值运算符将被隐式删除（因此程开发人员必须在需要时实现拷贝构造函数和拷贝赋值运算符）
* 如果类中没有定义移动构造函数和移动赋值运算符，且编译器没有生成默认的移动构造函数和移动赋值运算符，那么我们在代码中通过std::move()调用的移动构造或者移动赋值的行为将被转换为调用拷贝构造或者赋值运算符。

与拷贝操作一样，如果类中定义了移动操作，那么编译器就不会生成默认的移动操作，但是编译器生成移动操作的行为和生成拷贝操作的行为有些许不同，如下：

* 拷贝构造函数和拷贝赋值运算符是独立的：声明一个不会限制编译器生成另一个。所以如果你声明一个拷贝构造函数，但是没有声明拷贝赋值运算符，如果代码中用到了拷贝赋值操作，编译器会生成默认的拷贝赋值运算符。同样的，如果类中声明了拷贝赋值运算符但是没有拷贝构造函数，代码中用到拷贝构造函数时编译器就会生成默认的拷贝构造函数。该规则在C++98和C++11中都成立。
* 移动构造函数和移动赋值运算符不是相互独立的：如果类中声明了其中一个，编译器就不再默认生成另一个。所以显式声明移动构造函数会阻止编译器生成默认移动赋值运算符，显式声明移动赋值运算符同样会阻止编译器生成默认移动构造函数。如果类中声明了一个移动构造函数，就表明对于移动操作应怎样实现与编译器应生成的默认逐成员移动有些区别。如果逐成员移动构造有些问题，那么逐成员移动赋值同样也可能有问题。

#### std::move()函数
移动构造函数和移动赋值运算符的参数必须是右值引用。那么，对于一个左值，又如何使用移动语义呢？自C++11起，标准库提供了一个函数std::move()用于将左值转换成右值。

该函数在STL中定义如下：
```
template<typename _Tp>
constexpr typename std::remove_reference<_Tp>::type&& move(_Tp&& __t) noexcept { 
    return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); 
}
```
从上面定义可以看出，std::move()并不是什么黑魔法，而只是进行了简单的类型转换：
* 如果传递的是左值，则推导为左值引用，然后由static_cast转换为右值引用
* 如果传递的是右值，则推导为右值引用，然后static_cast转换为右值引用

使用std::move()之后，就意味着：
* 原对象不再被使用，如果对其使用会造成不可预知的后果
* 所有权转移，资源的所有权被转移给新的对象

std::move()函数并没有移动任何东西，它只是进行类型转换而已，真正进行资源转移的是类中实现的移动操作（移动构造函数和移动赋值运算符）。

#### 使用
自C++11起，开始支持右值引用。标准库中很多容器都支持移动语义，以std::vector<>为例，vector::push_back()定义了两个重载版本，一个像以前一样将const T&用于左值参数，另一个将T&&类型的参数用于右值参数。如下代码：
```
int main() {
    std::vector<BigObj> v;
    v.push_back(BigObj(10));
    v.push_back(BigObj(20)); 
    return 0;
}
```
两个push_back()调用都将解析为push_back(T&&)，因为它们的参数是右值。push_back(T&&)使用BigObj的移动构造函数将资源从参数移动到vector的内部BigObj对象中。而在C++11之前，上述代码则生成参数的拷贝，然后调用BigObj的拷贝构造函数。

如果参数是左值，则将调用push_back(T&):
```
int main() {
    std::vector<BigObj> v;
    BigObj obj(10);
    v.push_back(obj); // 此处调用push_back(T&)

    return 0;
}
```
对于左值对象，如果想要避免拷贝操作，则可以使用标准库提供的move()函数来实现(前提是类定义中实现了移动语义)，代码如下：
```
int main() {
    std::vector<BigObj> v;
    BigObj obj(10);
    v.push_back(std::move(obj)); // 此处调用push_back(T&&)

    return 0;
}
```

再看一个常用的函数swap()，在使用移动构造之前，我们定义如下：
```
template<class T>
void swap(T &a, T &b) {
    T temp = a; // 调用拷贝构造函数
    a = b;      // 调用operator=
    b = temp;   // 调用operator=
}
```
如果T是简单类型，则上述转换没有问题。但如果T是含有指针的复合数据类型，则上述转换中会调用一次复制构造函数，两次赋值运算符重载。

<div align=center><img src="/images/posts/cpp/move-semantic/1.png" width=""></div>

如果使用move()函数后，则代码如下：
```
template<class T>
void swap(T &a, T &b) {
    T temp = std::move(a);
    a = std::move(b);
    b = std::move(temp);
}
```
与传统的swap实现相比，使用move()函数的swap()版本减少了拷贝等操作。如果T是可移动的，那么整个操作将非常高效。如果它是不可移动的，那么它和普通的swap函数一样，调用拷贝和赋值操作，不会出错，且是安全可靠的。

<div align=center><img src="/images/posts/cpp/move-semantic/2.png" width=""></div>

#### 经验之谈
* 对于所有的基础类型（如int、double、指针以及其它类型），它们本身不支持移动操作(也可以说本身没有实现移动语义，毕竟不属于我们通常理解的对象)，所以，对于这些基础类型进行move()操作，最终还是会调用拷贝行为。
* 再移动构造函数或者移动赋值函数中，请务必将参数对象恢复默认值，否则，目标对象和原对象的底层资源指向同一个内存块，这样就导致原对象的析构函数和目标对象的析构函数会释放同一个内存块，进而导致程序崩溃。
* 不要在函数中使用std::move()进行返回。因为编译器可能对函数做(N)RVO优化，比移动语义效率更高。
* STL中大部分已经实现移动语义，比如std::vector<>，std::map<>等，同时std::unique_ptr<>等不能被拷贝的类也支持移动语义。