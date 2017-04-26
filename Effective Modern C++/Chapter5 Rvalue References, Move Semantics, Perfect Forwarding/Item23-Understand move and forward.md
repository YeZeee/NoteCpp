# Item23:Understand *std::move* and *std::forward*

C++11的一大特性就是引入了右值引用(*rvalue reference*)，移动语义(Move Semantics)与完美转发(Perfect Forwarding)就是由右值引用联结起来的：

- 移动语义(*move semantics*)：让编译器进行“移动”而不是“拷贝”操作来减少运行成本；同时也可以实现move-only类型。
- 完美转发(*perfect forwarding*)：使得函数模板能够“完美”的转发参数给内层函数。

但是注意移动语义并不移动、完美转发也并不完美、"type&&"也不代表右值。

## *std::move* and *std::forward*

C++11提供了*std::move*和*std::forward*，用于指示*move semantics*和*perfect forwarding*。  
但是*std::move*不移动任何东西、*std::forward*也不转发任何东西，甚至在运行时刻不做任何事情，不产生额外的代码。*std::move*和*std::forward*只是一个用于强制转换的函数模板。
- *std::move*无条件地将参数转换为右值
- *std::forward*只在特殊情况下，进行强制转换

## How *std::move* Works

*std::move*的C++11实现：

    template<typename T>
    stuct remove_reference {
        using type = T;
    };

    template<typename T>
    stuct remove_reference<T&> {
        using type = T;
    };

    template<typename T>
    stuct remove_reference<T&&> {
        using type = T;
    };

    template<typename T>
    typename remove_reference<T>::type&& move(T&& param) {
        using Type = typename remove_reference<T>::type&&;
        return static_cast<Type>(param);
    }

从实现可以看出，不论传入参数是右值还是左值(注意万能引用)，*std::move*通过*remove_reference*移除引用，然后强制转换得到右值引用作为返回值返回；重要的一点在于当右值引用作为函数值返回时，返回值是一个右值。  
C++14可以有更简便的实现：

    template<typename T>
    decltype(auto) move(T&& param) {
        using Type = remove_reference_t<T>&&;
        return static_cast<Type>(param);
    }

*std::move*其实是指示当前变量希望被进行*move*操作。  
这是一个支持从*std::string*构造的类，传入参数采用值传递，见Item41。

    class Annotation {
    public:
        explicit Annotation(std::string text): val(text);
        ...
    private:
        std::string val;
    }

当然传入参数应当是不变的：

    class Annotation {
    public:
        explicit Annotation(const std::string text): val(text) {};
        ...
    private:
        std::string val;
    }

然后我们希望能够支持从string中move而不是copy字符串:

    class Annotation {
    public:
        explicit Annotation(const std::string text): val(std::move(text) {};
        ...
    private:
        std::string val;
    }

这看起来没有问题，通过*std::move*强制转化为右值，调用*std::string*的移动构造函数，但其实不然：

问题的关键就在于text是一个const变量：

    class string {
    public:
        ...
        string(const string& rhs);
        string(string&& rhs);
        ...
    }

从*std::string*的构造函数可以看出，移动构造函数不能接收*const std::string&&*的右值，反而复制构造函数能够接受这样的右值，所以最终进行的是copy而不是move。从该例子可以看出：

- 如果希望使用*move operation*，就不要将变量声明为const。const的*move*最终匹配上的是*copy*。
- *std::move*只是进行了强制转换，指示该对象适合进行*move*，并不代表最终的操作是*move*。

## How *std::forward* Works

*std::move*是无条件的强制转换，而*std::forward*进行的是有条件的强制转换：

    template<typename T>
    constexpr T&& forward(typename remove_reference<T>::type& param) {
        // forward an lvalue as either an lvalue or an rvalue
        return static_cast<T&&>(param);
    }

    tempalte<typename T>
    constexpr T&& forward(typename remove_reference<T>::type&& param) {
        // forward an rvalue as an rvalue
        static_assert(!is_lvalue_reference<T>::value, "bad forward call");
        return static_cast<T&&>(param);
    }

关于参数的完美转发，即有如下函数：

    template<class T>
    void wrapper(T&& arg) 
    {
        // arg is always lvalue
        foo(std::forward<T>(arg)); // Forward as lvalue or as rvalue, depending on T
    }

参数arg的类型可以称作*forwarding reference*，这是*std::forward*实现完美转发的前提。
通过一层调用后arg对于内层函数总是一个左值，所以参数的完美转发使用前一个*std::forward*定义，但是模板参数T包含了原传入参数的信息实现完美转发：

- 当传入参数为rvalue，T总是为非引用类型，返回的将会是T&&，是一个rvalue。
- 当传入参数为lvalue，T总是为左值引用类型(包含const等限定)，返回的将会是T&，是一个lvalue。  

如下代码：

    void process(const Widget& lval);   // copy
    void process(Widget&& rval);        // move

    template<typename T>
    void UseProcess(T&& param) {
        process(std::forward<T>(param));
    }

    Widget w;
    UseProcess(w);          // use copy version.
    UseProcess(std::move(w));   // use move version.

## Things to Remember

- *std::move*只是将对象无条件强制转换为右值，并没有*move*。
- *std::forward*只是有条件的强制转换对象，并没有*forward*。
- *std::move*和*std::forward*在运行期没有作用，只是对编译的一种指示。