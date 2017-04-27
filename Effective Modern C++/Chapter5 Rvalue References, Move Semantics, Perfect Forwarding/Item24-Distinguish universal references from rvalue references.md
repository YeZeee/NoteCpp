# Item24:Distinguish Universal References From Rvalue References

在C++中，"&&"是具有迷惑性的：

    void f(Widget&& param);
    Widget&& var1 = Widget();
    template<typename T>
    void f(std::vector<T>&& param);
    
    auto&& var2 = var1;
    template<typename T>
    void f(T&& param);      

前3个声明，"&&"指代的是右值引用，而后两个并不是，它指代的是左值或者是右值，即万能引用(*universal reference* or *forwarding reference*)。它不仅可以绑定rvalue，也可以绑定lvalue；不仅可以绑定const，也可以绑定non-const；不仅可以绑定volatile，也可以绑定non-volatile。

## *Universal Reference* and Template Deduce

*Universal Reference*主要出现在两种情况，都属于类型推断：  
作为函数模板参数时：

    template<typename T>
    void f(T&& param);

作为auto推断时:

    auto&& var2 = var1;

*universal reference*的推断结果由initializer决定：当initializer是一个左值时，*universal reference*就是左值引用；当initializer是右值时，*universal reference*就是右值引用。

    Widget w;
    f(w);       // lvalue passed to f; param's type is Widget&.
    f(std::move(w));    // rvalue passed to f; param's type is Widget&&.

万能引用一定是与类型推断相关的，但是存在类型推断并不一定满足万能应用，引用的声明形式也是很重要的：

    template<typename T>
    void f(std::vector<T>&& param);     // rvalue reference.

    std::vector<int> v;
    f(v);        // error! cannot bind lvalue to rvalue reference.

该声明不满足T&&的形式，同时还有：

    template<typename T>
    void f(const T&& param);        // rvalue reference.

万能引用适配const和non-const的情况，上述参数声明也不是万能引用。

    template<class T, class Allocator = allocator<T>>
    class vector {
        ...
        void push_back(T&& x);

        template<class... Args>
        void emplace_back(Args&&... args);
    }

该形式，push_back看上去满足万能引用的形式，但是没有用到类型推断。当定义一个vector：

    std::vector<Widget> v;

类型推断出T为Widget，而push_back对应就有了实现:

    void push_bakc(Widget&& x);     // rvalue reference.

再emplace_back，仍然需要类型推断：

    template<class... Args>
    void emplace_back(Args&&... args);      // universal reference.

所以一个满足参数万能引用的函数模板有如下形式：

    template<typename T>
    void foo(T&& x);        // x is a universal reference.

## *Universal Reference* and Auto Deduce

auto类型推断也可以实现*universal reference*。因为auto的推断结果和模板的推断基本上一样，所以auto的*universal reference*有如下形式：

    auto&& ref = exp;

auto的万能引用不如模板的普遍，但是十分实用，特别是在C++14引入了lambda表达式的auto参数：

    auto FuncInvocation = 
        [](auto&& func, auto&&... params) {
            ...
            // Invoke func on params.
            std::forward<decltype(func)>(func)(std::forward<decltype(params)>(params)...);
            ...
        }

func是*universal reference*，params是*universal reference*的一个包；在函数内进行完美转发。

万能引用的背后其实是引用折叠(*reference collapsing*)机制在起作用，Item28。通过区分*rvalue reference*和*universal reference*使得代码更加具有抽象意义，减少定义模糊。

## Things to Remember

- *universal reference*有两种表达形式，分别在模板推断和auto推断中。
- 类型推断是*universal reference*的前提，不存在类型推断*type&&*就是*rvalue reference*。
- 当*universal reference*绑定左值时，结果为左值引用；当*universal reference*绑定右值时，结果为右值引用。