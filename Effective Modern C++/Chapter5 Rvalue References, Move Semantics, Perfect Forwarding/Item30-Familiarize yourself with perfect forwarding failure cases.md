# Familiarize Yourself With *perfect forwarding* Failure Cases.

完美转发意味着不仅仅转发对象，而且转发对象的值类型(左值还是右值？)，对象的cv-限定(const or volatile)。而*universal reference*的类型推断包含了对象的该类信息，帮助我们转发这些信息。

    template<typename T>
    void fwd(T&& param) {
        f(std::forward<T>(param));      // forward it to f.
    }

    template<typename... Args>
    void fwd(T&& param...) {
        f(std::forward<T>(param)...);   // forward package to f.
    }

但是完美转发并不总是成功转发的。比如fwd传入参数不符合f的要求就会导致转发的失败。还有一些参数碍于某些语言特性，不能够通过完美转发：

- 模板无法推断当前类型
- 编译器推断类型不符合内层函数的参数要求

## *Braced Initializer*

比如f有如下形式：

    void f(const std::vector<int>& v);

对f传入*braced initializer*是能够通过编译的：

    f({1, 2, 3});       // fine.

但是如果进行完美转发，结果是失败

    fwd({1, 2, 3});     // failure.

前者，*braced initializer*能够对参数vector<int>进行初始化。但是后者先要通过模板的类型推断，但是模板是无法对*braced initialzer*进行推断的(见Item2)。

使用*braced initializer*就属于编译器无法推断模板类型。

Item2中提到，auto可以对*braced initializer*进行推断，所以以下代码可以将*braced initializer*推断为*std::initialzer_list*通过编译。

    auto il = {1, 2, 3};
    fwd(il);        // fine.

## 0 or NULL as null Pointer

Item8解释了0和宏NULL和空指针的关系。对于模板而言，类型推断总是把0和NULL推断为整型(通常为int)。所以使用0和NULL也是无法完美转发的，属于第二种错误：

    void f(void* p);
    fwd(0);             // failure.
    fwd(NULL);          // failure.
    fwd(nullptr);       // fine.

## Declaration-only Integral *static const* and *constexpr* Data Member 

一般来说，不需要定义*static const*或者*static constepxr*的值，只需要声明就行。因为编译器会进行*const propagation*，从而不需要为这类变量开拓空间：

    class Widget {
    public:
        static const std::size_t MinVals = 28;  // declare, not define.
    }

对于调用该变量的函数，编译器会使用常量直接去补充参数：

    void f(std::size_t val);

    f(Widget::MinVals);     // fine.

但是对于完美转发就不行了，因为没有定义该变量，所以一切有关于对该变量的地址操作都是非法的。而完美转发模板对该参数的推断会是一个引用。

    fwd(Widget::MinVals);   // T is const size_t&. error should not link.

所以该操作是无法通过链接的。

但是这是对于标准而言，而许多编译器都会通过上述代码，但是这毕竟是不符合标准的，为了上述代码的可移植性，应当加入对该变量的定义:

    constexpr std::size_t Widget::MinVals;      // in cpp.

注意定义不应该重复初始化，初始化应该只存在于一个地方。

## Overload Function Names and Template Names

假如有以下接收函数指针为参数的函数：

    void f(int(*pf)(int));

当然可以忽略指针：

    void f(int(pf)(int));

有以下两个过程函数:

    int processVal(int val);        // ver.1
    int processVal(int val, int priority);  // ver.2

调用f:

    f(processVal);      // fine. use ver.1.

    fwd(processVal);    // error! which processVal？

要注意到完美转发模板终究只是一个模板，它不会顾及自己内部，只会先进行参数推断，再生成实例，才有内部实现。

解决这个问题，可以先将重载函数绑定给一个固定函数类型的指针上或者进行强制转化，再进行传参：

    int(*pf)(int) = processVal;
    fwd(pf);

    fwd(static_cast<int(*)(int)>(processVal));

## *Bitfields*

比如有以下位域定义：

    struct IPv4Header {
        std::uint32_t   version:4,
                        IHL:4,
                        DSCP:6,
                        ECN:2,
                        totalLength:16;
    }；

    void f(std::size_t sz);

    IPv4Header h;

    f(h.totalLength);       // fine. implicit conversion happen
    fwd(h.totalLength);     // error.

这个问题类似于*braced initializer*，模板无法对位域进行类型推断，同时没有任何引用和指针可以指向位域。所以解决方案就是将位域拷贝出来并做强制转换：

    auto length = static_cast<std::uint16_t>(h.totalLength);
    fwd(length);

## Things to Remember

- 完美转发通常在两种场景下失败：无法推断类型和“错误”推断类型。
- 常见的失败有：*braced initializer*、*0 or NULL as null pointer*、*declaration-only const static data member*、*template and overloaded funciton names*、*bitfields*