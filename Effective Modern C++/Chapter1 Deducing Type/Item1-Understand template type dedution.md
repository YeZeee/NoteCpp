# Item1: Understand Template Type Dedution


函数模板的形式如下:
    
    tmeplate<typename T>
    void foo(ParamType param);

函数调用的形式如下:

    foo(expr);  //call foo with expresstion;

在编译期间，编译器会推断T和ParamType。ParParamType通常会包含一些修饰符，比如const、&等，如下：

    template<typename>
    void foo(const T& param);

T的类型推断是由expr和ParamType共同决定的。如下根据ParamTpye分成三种情况：

- ParamType是一个指针和引用（非万能引用）。
- ParamType是一个万能引用。
- ParamType不是引用和指针。

## case1: ParamType is a Reference or Pointer,but not a Universal Reference

在这种情况下,类型推断步骤如下：
- 如果expr是一个引用，则忽略引用部分。
- 依赖ParamType模式匹配expr的类型，得到T。

比如：

    template<typename T>
    void foo(T& param);
    
    int x = 27;
    const int cx = x;
    const int& crx = x;

    foo(x);     //T is int, ParamType is int&.
    foo(cx);    //T is const int, ParamType is const int&.
    foo(crx);   //T is const int, ParamType is const int&. reference-ness is ignored.

这种情况下，为了保证const变量的安全，const将成为T的一部分。如果param包含const修饰，如下：

    template<typename T>
    void foo(const T& param);

    foo(x);     //T is int, ParamType is const int&.
    foo(cx);    //T is int, ParamType is const int&.
    foo(crx);   //T is int, ParamType is const int&. reference-ness is ignored.

param为指针的表现与引用类似，如下：

    template<typename T>
    void foo(T* param);
    const int* cpx = &x;

    foo(&x);    //T is int, ParamType is int*.
    foo(&cx);   //T is const int, ParamType is const int*.
    foo(cpx);   //T is const int, ParamType is const int*.

## case2: ParamType is a Universal Reference

在这种情况下，推断原则如下：

- 如果expr是一个左值，T和ParamType都被推断为左值引用。
（1.只有这种情况，T才有可能被推断为引用；2.ParamType被推断为左值引用而非右值引用。）
- 如果expr是一个右值，直接适用通用情况。

比如：

    template<typename T>
    void foo(T&& param);    //ParamType is a Universal Reference.

    foo(x);     //T is int&, ParamType is int&.
    foo(cx);    //T is const int&, ParamType is const int&.
    foo(crx);   //T is const int&, ParamType is const int&.
    foo(27);    //27 is int and rvalue, T is int, ParamType is int&&.

当ParamType是万能引用时，expr左值和右值表现并不一样。

## case3： ParParamType is Neither a Pointer nor a Reference

按值传递,这意味着param将是传入的一个copy。原则如下：

- 如果expr是一个引用，则忽略引用。
- 如果忽略引用后，有const和volatile修饰，则亦忽略。

如下：
    
    template<typename T>
    void foo(T param);      //param is passing by value.

    foo(x);     //T is a int, ParamType is a int.    
    foo(cx);    //T is a int, ParamType is a int. const-ness is ignored.
    foo(crx);   //T is a int, ParamType is a int. reference-ness and const-ness is ignored.

const被忽略的原因在于：param是一个copy，被copy对象不可变不代表其复制不可变。
这是按值传递和引用或指针传递不同的地方。
考虑以下情况：

    const char* const ptr = ... //ptr is a const pointer pointing to const.
    foo(ptr);   //T is a const char*, ParamType is a const char*. 
                //top-level is ignored. but low-level should not be ignored.

由于按值传递的对象是指针。所以修饰指针本身的const（top-level）可以被忽略，
但是修饰指针所指向的obj的const不可以被忽略。

## Array Argments

值得注意数组和指针是不一样的，即时它们经常可以互相转换。
数组经常退化为指针（指向第一元素）：

    const char name[] = "abcd";     //name's type is const char[13].
    const char *ptr = name;         //array decays to pointer.

考虑数组作为expr的情况：

    template<typename T>
    void foo(T param);      //by-value.

    foo(name);  //T is const char*, param is const char*.
                //array decays to pointer.

    template<typename T>
    void foo(T& param);      //by-reference.

    foo(name);  //T is const char[5], ParamType is const char(&)[5].

    template<typename T>
    void foo(T* param);      //by-pointer.

    foo(name);  //T is const char, ParamType is const char*.

通过该特性，就可以实现以下模板：

    //aquire the size of an array at compile time.
    template<typename T, size_t N>
    constexpr size_t ArraySize(T (&)[N]) noexcept {
        return N;
    }

## Function Argments

函数也可以退还成函数指针。模板类型推断时的函数与数组表现类似。
如下：

    void func(int,double);   //type is void(int,double).

    template<typename T>
    void foo(T param);       //by-value.

    foo(func);  //T is void(*)(int,double), ParamType is void(*)(int,double).

    template<typename T>
    void foo(T param);      //by-reference.

    foo(func);  //T is void(int,double), ParamType is void(&)(int,double).

    template<typename T>
    void foo(T* param);      //by-pointer.

    foo(func);  //T is void(int,double),ParamType is void(*)(int,double).


## Things to Remember

- 在模板类型推断中，argument的reference-ness被忽略。
- 对于universal reference parameter，lvalue arguments是具有特殊机制的。
- 对于by-value parameter，const和volatile是被忽略的。
- 对于array和funciton的arguments，除了reference parameter以外均退化为指针。


