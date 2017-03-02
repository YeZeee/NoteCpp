# Item4: Konw How to View Deduced Type

程序开发过程中，总共有3个阶段，会对推断类型感兴趣。

- during edit your code.
- during compilation.
- at runtime.

## During Edit Your Code

通常IDE能够直接告诉你当前类型推断是什么，但是这往往针对类型比较简单的情况下。
IDE能够显式这类信息是，其内部的compiler正在运行。如果这个compiler对当前情
况进行足够的语法解析。那么IDE就难以给出这类信息。
简单类型推断往往是快速准确的，但是涉及到比较复杂的推断，这种方式往往难以给出
有效的信息。

## During Compilation

在编译过程中，可以通过故意制造编译错误，查看错误报告来获得推断结果。

如下：

    template<typename T>
    class TD;               //TD means "Type Display".

这是一个未定义的模板，对该模板的实例化会导致编译时错误。

比如：

    int x;
    const int* y = &x;
    TD<decltype(x)> xType;
    TD<decltype(y)> yType;

编译器报告编译错误：

>error C2079: 'xType' uses undefined class 'TD<int>'

>error C2079: 'yType' uses undefined class 'TD<const int *>'

## At Runtime

typeid与type_info支持程序识别自身的变量类型。
如下:

    std::cout<< typeid(x).name() << std::endl;
    std::cout<< typeid(y).name() << std::endl;

于VS环境下可以获得：    
>int

>int const *

但是事情并没有这么简单，考虑以下例子：

    template<typename T>                //function to show T and param.
    void f(const T& param);

    std::vector<Widget> createVec();    //factory function.

    const auto vw = createVec();        

    if(!vw.empty()){
        f(&vw[0]);
    }

然后我们通过f来获得上面类型推断的结果：

    template<typename T>
    void f(const T& param){
        std::coud << "T = " << typeid(T).name() << '\n';
        std::cout << "param = " << typeid(param).name() << '\n';
    }

在VS中得到的结果如下：
>T = class Widget const *

>param = class Widget const *

很显然T和param推断出的类型应该是不一样的，因为param是const T&。
所以type_info::name并不可靠，
这是因为标准要求type_info::name对模板函数传入均使用by—value的方式。

按照Item1，vw[0]的类型是一个const Widget&，
&vw[0]的类型便是const Widget *,
按值传递类型推断出T和ParamType均为const Widget *。

>此外还可以使用boost库


## Things to Remember

- Deduced Types可以通过IDE，编译错误，以及type_info和Boost TypeIndex library查看.
- 结果可能并不是那么有效或者精确的，所以Item1的内容是至关重要的。
