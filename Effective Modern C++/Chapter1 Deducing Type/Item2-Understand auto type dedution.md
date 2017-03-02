# Item2: Understand Auto Type Dedution


Auto的类型推断与item1的模板类型推断基本一致。在auto推断中，auto相当于
模板中的T，整个类型修饰符相当于PramaType，初始化的值相当于expr。(有一点不同)

如下：

    auto x = 27;

    template<typename T>
    void funcx(T param);
    
    funcx(27);      //T is int, so auto is int. ParamType is int.

    const auto cx = x;
    
    template<typename T>
    void funccx(const T param);

    funccx(x);      //T is int, so auto is int. ParamType is const int.
    

    const auto& crx = x;

    template<typename T>
    void funccrx(const T& param);

    funccrx(x);     //T is int, so auto is int. ParamType is const int&.

## Type Specifiers have effect on Type Deduction 

正如模板类型推断中的以PramaType分类讨论，auto类型推断可以将type specifier分成3类讨论，即：

- 类型修饰符是引用或者指针（不包含万能引用）
- 类型修饰符是万能引用
- 类型修饰符不是指针和引用

比如：

    auto x = 27；            //27 is int, x is int.
    const auto cx = x;     //x is int, cx is const int. 
    const auto& crx = x;    //x is int, crx is const int&.

    auto&& urx = x;         //x is int(lvalue), urx is int&.
    auto&& urx = cx;        //cx is const int(lvalue), urx is const int&.
    auto&& urx = 27;        //27 is rvalue, urx is int&&.


## Initializer is an Array or a function

指针和数组作为初始化auto类型推断和模板类型推断一致。

    const char name[] = "123456";   //name is const char[7].

    auto arr1 = name;       //by-value, arr1 is const char*.
    auto& arr2 = name;      //by-lreference, arr2 is const char(&)[7].

    void func(int,double);          //func is void(int,double).

    auto func1 = func;      //by-value, func1 is void(*)(int,double).
    auto& func2 = func;     //by-lreference, func2 is void(&)(int,double).

## Iraced-Initializer and Parenthesis-Initializer

barced-initializer_list是auto和template类型推断的唯一不同之处。template是不能直接推断出initializer-list的。
考虑以下定义：

    auto x1 = 27;           //type is int, value is 27.
    auto x2(27);            //ditto.
    auto x3 = { 27, 42 };   //type is initializer_list<int>, value is {27}.
    auto x4{ 27 };          //type is int, value is 27.
    auto x5{ 27, 42 };      //error, need =.
    auto x6{ 1, 2, 3.0 };   //error, cannot deduce T for initializer_list.

注意直接初始化和赋值初始化对于{}initializer的区别。

    template<typename T>
    void foo(T param);
    
    foo({ 1, 2, 3 });       //error, connot deduce type for T.

想要实现对T的template推断，可以如下声明：

    template<typename T>
    void foo(std::initializer_list<T> inilist);

    f({1, 2, 3});           //valid, T is int, ParamType is initializer_list<int>.

同时值得注意的是，C++14允许函数返回值与lambda表达式参数通过使用auto进行类型推断，但是这种情景下的类型推断是模板类型推断，
而不是auto类型推断，比如：

    auto createList() {
        return { 1, 2, 3 };     //error: cannot deduce return type for { 1, 2, 3 }.
    }

    std::vector<int> v;
    auto resetV = [&v](const auto& newvalue) {
        v = newvalue;           //C++ 14;
    }
    resetV({ 1, 2, 3 });        //error! cannot deduce parameter type for { 1, 2, 3 }.

## Things to Remember

- auto类型推断和template类型推断几乎一致，只是auto可以将braced-initializer推断为initializer_list，而template不接受这种推断。
- auto在函数返回值和lambda表达式参数使用时，代表的是template推断，而不是auto。