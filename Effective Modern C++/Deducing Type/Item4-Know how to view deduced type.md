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




