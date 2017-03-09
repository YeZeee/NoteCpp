# Item9: Prefer Alias Declarations to typedefs

别名是C++减少长类型名拼写的有效手段。  
在C++98中：


    typedef std::unique_str<std::unordered_map<std::string, std::string>> UPtrMapSS;

而在C++11中：

    using UPtrMapSS = std::unique_str<std::unordered_map<std::string, std::string>>；

以上两例，无法看出typedef和using的区别与差距，也看不出using的优势。  
但是当我们想为某些类型，比如函数指针、数组等起别名时，语义的表达性就不一样了：

    typedef void (*fp)(int, const std::string&);     // typedef.

    using fp = void(*)(int, const std::string&);     // alias declaraiton.

显然using对于fp的表达更加直观，但这仍然不是alias declaration优于typedef的绝对理由。

## About Template Alias

using可以用于template(alias templates)，而typedef不可以。  
传统的将别名用于模板的做法是为typedef加上一层struct的封装：

    template<typename T>
    typedef std::list<T, MyAlloc<T>> MyAllocList;   // error! canot typedef.

    template<typename T>
    struct MyAllocList {
        typedef std::list<T, MyAlloc<T>> type;      // Add struct outside. 
    }

    MyAllocList<Widget>::type lw;           // Client code.

    template<typename T>
    using MyAllocList = std::list<T, MyAlloc<T>> ;   // Fine.

    MyAllocList<Widget> lw;                 // Client code.

using与typedef在用户代码上的差距就表现出来了。除此之外，还有一个using优于typedef的方面
，当我们在template中使用'typedef型'的模板别名时，::type是dependent type，编译器无法知道这是不是一个类型，就必须加typename来指示该名为类型成员：

    template<typename T>
    class Widget {
    private:
        typename MyAllocList<T>::type list;
    }


而using不需要外部封装，就没有这方面的问题：

    template<typename T>
    class Widget {
    private:
        MyAllocList<T> list;
    }

MyAllocList是一个类型别名，所以MyAllocList<T>必须是一个类型，而不是变量，所以MyAllocList<T>是non-dependent type，所以typename是可以忽略的。  
再看以下代码：

    class Wine { ... };

    template<>                      // A specialization of MyAllocList.
    class MyAllocList<Wine> {       // when T is wine.
    private:
        enum class WineType
        { White, Red, Rose };       // Here type is data member, not type member.
        WineType type;
        ...
    }

可以看到这里type是一个数据成员，而不是类型成员。如果Widget对T = Wine生成一个实例。那么::type
在Widget模板中就是一个数据名。这就是为何需要typename的原因。

C++11中提供了一系列类型处理的功能模板(base on TMP),<type_traits>

    std::remove_const<T>::type              // Yields T from const T.
    std::remove_reference<T>::type          // Yields T from T&.
    std::add_lvalue_reference<T>::type      // Yields T& from T.

这些模板的实现都是依赖于内嵌typedef的。

C++14给了更好的实现，依赖于using:

    std::remove_const_t<T>              // Yields T from const T.
    std::remove_reference_t<T>          // Yields T from T&.
    std::add_lvalue_reference_t<T>      // Yields T& from T.

    template<typename T>
    using remove_const_t = typename remove_const<T>::type;

    template<typename T>
    using remove_reference_t = typename remove_reference<T>::type;
    
    template<typename T>
    using add_lvalue_reference_t = typename add_lvalue_reference<T>::type;
    
## Things to Remember

- typedefs不支持模板化，但是using支持。
- alais template避免了嵌套和::type的后缀，注意typename对于dependent type的作用。
- C++14提供了traits的更好的实现。





